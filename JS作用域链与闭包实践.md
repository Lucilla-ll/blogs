JS 作用域链与闭包实践

**目标需求**

实现一个立即执行函数，收集网页相关参数通过接口发送到第三方（fb）

**设计实现**

```js
// 主工具对象
const facebookCAPI = {
	// 初始化配置
    init: function (options) {
        Object.assign(config, options);
        if (config.autoTrackPageView) {
            this.waitForFbp(function (fbp) {
                if (fbp) {
                    this.trackEvent("PageView");
                } else {
                    console.warn("cannot get _fbp Cookie");
                }
            });
        }
    },
    
    waitForFbp: function (callback, maxAttempts = 10, interval = 100) {
        let attempts = 0;
        const checkInterval = setInterval(() => {
            attempts++;
            const fbp = this.getCookie("_fbp");
            if (fbp || attempts >= maxAttempts) {
                clearInterval(checkInterval);
                callback(fbp);
            }
        }, interval);
    },
    
    //跟踪事件
    trackEvent: function (eventName, customData = {}) {}
}
```

**问题**

```js
this.waitForFbp(function (fbp) {
    if (fbp) {
    	// error：trackEvent is undefined, 这里的 this 不再是 facebookCAPI 对象
    	this.trackEvent("PageView");
    } else {
        console.warn("cannot get _fbp Cookie");
    }
});
```

**问题分析**

- 当函数作为回调传递时，它的 `this` 会丢失原始绑定
- 在这个匿名回调函数中，`this` 可能指向全局对象（浏览器中是 `window`）或 `undefined`（严格模式）



:white_check_mark:**解决方案1**

```js
this.waitForFbp((fbp) => { // 改为箭头函数
    if (fbp) {
        this.trackEvent("PageView"); // 现在 this 能正确指向
    }
});
```

:white_check_mark:**解决方案2**

```js
this.waitForFbp(function(fbp) {
  if (fbp) {
    this.trackEvent("PageView");
  }
}.bind(this)); // 显式绑定 this
```

:white_check_mark:**解决方案3**

```js
const self = this;
this.waitForFbp(function() {
  self.trackEvent("PageView"); // 通过闭包保留 this
});
```

:white_check_mark:**解决方案4**

```js
this.waitForFbp(function(fbp) {
  if (fbp) {
    facebookCAPI.trackEvent("PageView"); //facebookCAPI 是定义在闭包中的常量
  }
});
```

**解决方案4 分析**

闭包：代码被包裹在 **IIFE（立即调用函数表达式）** 中：

```js
(function(window, document) {
  "use strict";
  const facebookCAPI = { ... };  // ← 这里定义的对象
  window.facebookCAPI = facebookCAPI; // 暴露到全局
})(window, document);
```

- **`facebookCAPI` 是一个常量**，在 IIFE 的函数作用域内定义
- 内部的所有函数（如 `init`）都**共享这个作用域**，因此可以直接访问 `facebookCAPI` 变量



JavaScript 的变量查找规则：

```js
init: function(options) {
  // 这里可以访问 facebookCAPI，因为：
  // 1. 先在当前函数作用域查找 → 未找到
  // 2. 向外层作用域（IIFE）查找 → 找到 const facebookCAPI
  // 这里的 this ≠ facebookCAPI！
  // 但 facebookCAPI 仍可通过作用域链访问
  facebookCAPI.trackEvent("PageView"); 
}
```

- 内部函数可以访问外部函数的变量（**词法作用域**）
- 即使 `this` 指向改变，通过作用域链仍能访问到 `facebookCAPI`



PS: If we want to retrieve `document.cookie`, we can't simply open the `.html` file directly in the browser. Instead, we need to use a VS Code plugin called **Live Server**.
