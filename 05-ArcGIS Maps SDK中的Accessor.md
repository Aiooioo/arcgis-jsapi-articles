Accessor是ArcGIS Maps SDK for JavaScript中绝大多数类型的基类。通过Accessor类型的封装，为SDK中提供了响应式类型的基础。

Accessor由多个不同层次的代码所组成，并设计了复杂的逻辑结构，提供除响应式更新之外的多种不同的特性，让我们从用户所接触的Accessor类开始。

#### esri/core/Accessor
---
位于esri/core（或者@arcgis/core/core）命名空间中的Accessor类型实际并不直接提供响应式的能力，他是一层包装类，目的是提供一致的外层API，以及定义生命周期，这里实际上是一个适配器，实现了OO编程中经典的适配器模式来隔离变更。SDK在当前4.28版本之前的多个版本中实际对于底层的Accessor实现进行了多次的优化，乃至于重构，但对于使用者而言，这完全是无感的。

```JavaScript
class Accessor {
	static createSubclass(args) {}

	constructor(...args) {}

	postscript(ctorArgs) {}

	initialize() {}

	destroy() {}

	commitProperty(p) {}
	get(path) {}
	set(path, value) {}

	hasOwnProperty(p) {}
	keys() {}

	watch(path, handler, opts) {}
	addHandles() {}
	removeHandles(h) {}
	removeAllHandles() {}
	hasHandles() {}

	notifyChange(property) {}
}
```

如果我们将以上的Accessor类型接口定义进行分组，我们可以梳理得到Accessor所承担的3个职责，分别是：
1. 生命周期定义：`constructor`、`postscript`、`initialize`、`destroy`
2. 响应式代理/适配器：`get`、 `set`、 `commitProperty`
3. 观察者模型（类似RxJS中的Subject）：`watch`、`notifyChange`

另外，新版本中为了更好的管理Accessor对象，还额外为Accessor类型新增了状态机制。

这就是ArcGIS Maps SDK for JavaScript中强大的Accessor。

我们从构造函数开始深入了解Accessor类型。

```JavaScript

const accessorHandlesSymbol = Symbol("Accessor-Handles")
const accessorInitialized = Symbol("Accessor-Initialized")

class Accessor {
	constructor(...args) {
		// Step 1: 初始化状态机
		this[accessorHandlesSymbol] = null;
		this[accessorInitialized] = false;

		// 禁止直接创建Accessor基类型的实例
		if (this.constructor === Accessor) {
			throw new Error("[accessor] cannot instantiate Accessor. This can be fixed by creating a subclass of Accessor");
		}

		// Step 2: 创建响应式属性代理对象
		const properties = new esri.core.accessorSupport.Properties(this);
		Object.defineProperty(this, "__accessor__", { 
			enumerable: !1, 
			value: properties,
		})


		// Step 3: 为继承类型预留的统一参数格式化处理接口
		if(args.length > 0 && this.normalizeCtorArgs) {
			properties.ctorArgs = this.normalizeCtorArgs.apply(this, args);
		}
	}
}
```

由于Accessor作为基类，预定义了大量的空方法，约等同于Java等高阶语言中的抽象类，不允许直接实例化。作为统一的API，在Accessor类型之上提供了静态的函数继承入口。

```JavaScript
class Accessor {
	static createSubclass(typeProps = {}) {
		/* 在旧版本中，Accessor曾经基于dojo来定义，因此能支持多重继承
		*  随着新版本完全移除了dojo，为了兼容TypeScript的类型机制，不再允许多种继承
		*/
		if (Array.isArray(typeProps)) {
			throw new Error("Multi-inheritance unsupported since 4.16");
		}

		// 解构类型描述中关于继承类的定义型属性并与其他属性剥离
		const { properties: t, declaredClass: e, constructor: s } = typeProps;
		delete r.declaredClass;
		delete r.properties;
		delete r.constructor;

		// 生成继承类型
		const o = this;
		class i extends o {  
		  constructor(...r) {  
			super(...r);
			(this.inherited = null);
			 s && s.apply(this, r);  
		  }  
		}

		// 解析继承类型属性元信息
		getPropertiesMetadata(i.prototype);

		for (const c in r) {
			// 逐个解析传入的其他属性元信息
			// 此处省略
		}

		for (const c in t) {
			// 解析初始化传递的属性定义并构建Property代理对象
			// 此处省略
		}

		return subclass(e)(i)
	}
}
```

`subclass`作为一个类装饰器，为继承的子类型清扫了JavaScript语言层面的最后障碍，并补充了类型定义关键信息，最后将属性元信息`__accessorMetadata__`与属性代理`__accessor__`进行关联，构成了Accessor类型的最后一块拼图。

让我们继续深入了解Accessor如何构建SDK的基础。

