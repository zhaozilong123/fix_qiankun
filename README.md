# 如何修复qiankun子应用切换样式闪烁

在使用qiankun的过程中，我们发现在已经加载过一次的子应用直接互相切换，会有一瞬间样式丢失，经过查找qiankun GitHub仓库的issue，发现有大量一样的场景，但是官方也没有一个比较好的解决方案提出。于是把qiankun包拿下来学习修复，最终解决了这个问题。这篇文章主要是分享qiankun对于样式缓存的策略还有自己解决过程的一些思路。

---

## qiankun子应用如何缓存

### 缓存流程

[乾坤源码分析](https://github.com/a1029563229/blogs/blob/master/Source-Code/qiankun/1.md)

---

### 如何动态记录样式

qiankun对js动态使用appendChild插入样式标签做了统一的拦截，目的是为了在之应用里面动态加载的资源放到子应用自己所在的容器，方便在卸载的时候卸载掉加载的资源。关键代码如下：

```
// 对样式拦截函数
const rawHeadRemoveChild = HTMLHeadElement.prototype.removeChild;
const rawBodyAppendChild = HTMLBodyElement.prototype.appendChild;
const rawBodyRemoveChild = HTMLBodyElement.prototype.removeChild;
const rawHeadInsertBefore = HTMLHeadElement.prototype.insertBefore;
const rawRemoveChild = HTMLElement.prototype.removeChild;
```

动态创建的样式元素会存到一个名字叫dynamicStyleSheetElements的数组里面：

```
export type ContainerConfig = {
  appName: string;
  proxy: WindowProxy;
  strictGlobal: boolean;
  // 这里是关键存放动态样式元素的数组
  dynamicStyleSheetElements: HTMLStyleElement[];
  appWrapperGetter: CallableFunction;
  scopedCSS: boolean;
  excludeAssetFilter?: CallableFunction;
};
```

---

### 缓存的形态以及如何记录

在子应用卸载的时候会把样式缓存都存到一个WeakMap结构里面，存的是样式标签的 **【sheet】** 属性的 **【cssRules】** 。

```
// 样式缓存的WeakMap结构
const styledComponentCSSRulesMap = new WeakMap<HTMLStyleElement, CSSRuleList>();
...
// 在子应用卸载的时候执行的记录样式方法
export function recordStyledComponentsCSSRules(styleElements: HTMLStyleElement[]): void {
  styleElements.forEach((styleElement) => {
    if (styleElement instanceof HTMLStyleElement && isStyledComponentsLike(styleElement)) {
      if (styleElement.sheet) {
        styledComponentCSSRulesMap.set(
          styleElement, (styleElement.sheet as CSSStyleSheet).cssRules
        );
      }
    }
  });
}
```

---

### 缓存的读取

如果我们的子应用有动态加载的样式，比如使用了react.lazy，虽然qiankun对js已经做了缓存，但是js去动态创建样式还是会每次都执行，每次动态创建的样式都会在2.2中描述拦截记录一次，然后在已经缓存过的样式缓存Map里面去查询读取，最终使用样式元素的【insertRule】方法去插入样式：

```
// 获取缓存的函数，从2.3描述的Map结构里面读取缓存
export function getStyledElementCSSRules(styledElement: HTMLStyleElement): CSSRuleList | undefined {
  return styledComponentCSSRulesMap.get(styledElement);
}

...

export function rebuildCSSRules(
  styleSheetElements: HTMLStyleElement[],
  reAppendElement: (stylesheetElement: HTMLStyleElement) => boolean | HTMLStyleElement,
) {
  styleSheetElements.forEach((stylesheetElement) => {
   	// reAppendElement 函数就是在子应用重新加载的时候又动态去创建，创建成功会返回true
    const reAppendResult = reAppendElement(stylesheetElement);
    if (reAppendResult) {
     if (stylesheetElement instanceof HTMLStyleElement && isStyledComponentsLike(stylesheetElement)) {
        const cssRules = getStyledElementCSSRules(stylesheetElement);
        if (cssRules) {
          for (let i = 0; i < cssRules.length; i++) {
            const cssRule = cssRules[i];
            const cssStyleSheetElement = stylesheetElement.sheet as CSSStyleSheet;
            // 把缓存的cssrule插入到新创建的标签
            cssStyleSheetElement.insertRule(cssRule.cssText, cssStyleSheetElement.cssRules.length);
          }
        }
      }
    }
  });
```

---
## 源码问题&解决

这里要预先知道一些浏览器的一些css渲染机制，了解的可以忽略。浏览器对加载过的样式会有缓存，但是人为的把相同href的link标签删除再插入会导致样式表丢失，会从缓存再次读取然后生成CSSOM树，合成的样式表都放在样式的【sheet】属性里面。参考官方文档：[https://www.w3.org/TR/cssom-1/#associated-css-style-sheet](https://www.w3.org/TR/cssom-1/#associated-css-style-sheet)

### 问题1：缓存结构的问题

qiankun使用的weakMap，把样式元素作为一个Map的key去存，但是每次拦截创建都会是新的元素，那么就会导致读取不到缓存的情况。于是我又新添加了一个缓存，用link标签的href去作为key缓存样式表，解决如下：

```
// fix: 添加一个以link的href为key的缓存
const linkStyleRulesMap = new Map<string, CSSRuleList>();
```

### 问题2：样式资源跨域拿不到sheet属性

经过测试，如果静态资源没有加 【crossorigin="anonymous"】属性，会导致拿不到sheet属性，那么缓存当然就无法设置无法读取了，这个qiankun文档也没有详细的描述。解决如下：

```
// 子应用 webpack 配置
{
  ...
	output: {
    ...,
    crossOriginLoading: 'anonymous',
  }
  ...
}
```

### 问题3：问题1、2解决了，发现创建已经加载过的相同href的Link标签还是不能把缓存的cssrule赋值过去

这个可以看一下第3点的提示，浏览器在不刷新的情况下，对于已经加载过的相同href的link标签，卸载的时候会把sheet属性卸载掉，但是重新创建这个属性会是只读的。这里就需要需去曲线解决一下，只要是新创建的style标签，我们是可以去操作它的【sheet】属性的，缓存插入创建的空style标签后把空标签插入到dom中,解决如下：

```
function rebuildCSSRules(
	styleSheetElements: HTMLStyleElement[],
  reAppendElement: (stylesheetElement: HTMLStyleElement) => boolean | HTMLStyleElement,
){
	...
	if(stylesheetElement.getAttribute('href') && stylesheetElement.getAttribute('crossorigin') === 'anonymous'){
    const cssRules = getStyledElementCSSRules(stylesheetElement);
    if (cssRules) {
      // eslint-disable-next-line no-plusplus
      for (let i = 0; i < cssRules.length; i++) {
        const cssRule = cssRules[i];
        const cssStyleSheetElement = rebuildElement.sheet as CSSStyleSheet;
        cssStyleSheetElement.insertRule(cssRule.cssText, cssStyleSheetElement.cssRules.length);
      }
    }
  }
  ...
}

function rebuild() {
   // fix: 如果是link标签并且有href属性，使用字符串缓存，缓存过的style对象卸载的时候把sheet对象给自动去除了
  rebuildCSSRules(dynamicStyleSheetElements, (stylesheetElement) => {
    const appWrapper = appWrapperGetter();
    if (!appWrapper.contains(stylesheetElement)) {
      if(stylesheetElement.getAttribute('href') && stylesheetElement.getAttribute('crossorigin') === 'anonymous'){
        // 动态创建一个空的style标签去作为缓存读取的容器
        const linkElement = document.createElement('style');
        linkElement.setAttribute('type','text/css');
        rawHeadAppendChild.call(appWrapper, linkElement);
        // 插入原始无sheet属性标签（为了防止自定义style标签插入两次）
        rawHeadAppendChild.call(appWrapper, stylesheetElement);
        return linkElement;
      }else{
        rawHeadAppendChild.call(appWrapper, stylesheetElement);
        return true;
      }
    }

    return false;
  });
};
```

如下示意图：上面的空标签虽然没有内容，但实际上是读取了缓存的样式表存放的容器，是真正读取样式缓存渲染的地方，下面那个是实际资源插入以方便以防万一另一方也防止多次插入，可以理解为上面空标签的快照。
![](https://km.sankuai.com/api/file/cdn/1219957489/1333798004?contentType=1&isNewContent=false&isNewContent=false)



