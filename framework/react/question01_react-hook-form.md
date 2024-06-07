# react-hook-form 源码解析

在前端领域中，表单是随处可见的，是页面与用户之间进行数据交互的主要载体。最开始的时候并没有表单组件的概念，表单内每个字段自己维护自己的状态，并没有聚拢在一起管理。但是随着表单需求和交互的复杂度慢慢变大，原始的方式让开发者在使用表单的时候非常痛苦。之后随着前端组件的概念产生，开发者们抽象出了表单组件来处理表单数据管理的问题。

表单组件的功能大致可以分成三个方面：

- 值收集
- 值校验
- 值更新

虽然看起来只有这三个方面的内容，但是其中还伴随着表单组件多个状态的切换，处理起来是非常繁琐和复杂的，令人头秃！

## 1. 基于React框架的比较流行的表单组件

react中常用的表单组件：

- [Formik](https://github.com/formium/formik) — 负责管理表单的数据，没有提供UI层面的支持，表单字段的值采用受控组件的方式来管理。
- [Antd](https://github.com/ant-design/ant-design) 中的Form组件 — 依托于整个antd组件库，不仅管理表单的数据，还提供了一套标准UI的字段组件。同样内部也是使用受控组件的方式来管理字段的值。
- [react-hook-form](https://github.com/react-hook-form/react-hook-form) — 也是一个针对表单数据层面管理的组件，采用的是react-hook风格的api，内部使用非受控组件的方式来管理字段的值

对比上面三个比较常见的方案，各有各的特点：

1. Antd 的表单方案很完整，不仅有表单数据层面的管理，还提供了标准UI的字段组件，使用起来比较方便。但是相对耦合比较严重，而且内部使用受控组件，组件重新渲染的次数过多，导致性能下降。
2. Formik 只负责表单数据层面的管理，灵活性比较好。但是Formik提供的是**包裹式**的API，也就是需要在组件外部再包裹一个组件。而且内部也是使用受控组件，组件重新渲染的次数过多，导致性能下降。
3. react-hook-form 也只负责表单数据的统一管理，UI层面由开发者来自己解决。react-hook-form采用hook API，**嵌入式**的在目标组件中插入数据管理的逻辑。而且内部使用非受控组件，极大程度的减少了组件重新渲染的次数，性能更佳。

## 2. react-hook-form源码实现的一些点

### 2.1 数据管理和组件刷新完全分开

一般表单组件的实现中，都会用 useState 来构建一个状态来管理表单的数据，这样一来每次数据变化的时候就能及时刷新组件，展示组件的最新状态，配合受控组件使用刚好契合。

但是这种方式也有一个比较致命的**缺点**，就是受控组件在输入的时候，会频繁的修改state，也就是说会导致组件不断地刷新，不仅是刷新字段本身，还会刷新同一表单中的其他字段，造成无意义的消耗，降低性能。

React-hook-form 抛弃掉了这种方式，将数据管理和组件刷新分开，数据变化的时候并不会自动导致组件刷新。

#### 2.1.1 使用 useRef 配合非受控组件管理表单数据

源码中表单的值是使用useRef来统一管理的：

```js
/**
 * 存储表单字段的值
 * 每个字段的基本结构是{ref:Ref,...（字段的其他属性）}，会存储整个DOM对象
 */
const fieldsRef = React.useRef<FieldRefs<TFieldValues>>({});

/**
 * 存储表单字段的异常信息
 */
const errorsRef = React.useRef<FieldErrors<TFieldValues>>({});

...
```

内部的状态基本上都是通过useRef来管理的。然后通过非受限组件的方式来收集和更新字段值。

```js
// 收集字段的值
function registerFieldRef < TFieldElement extends FieldElement < TFieldValues >> (
    ref: TFieldElement & Ref,
    validateOptions: ValidationRules | null = {},
): ((name: InternalFieldName < TFieldValues > ) => void) | void {
    // ...
    const fieldRefAndValidationOptions = {
        ref,
        ...validateOptions,
    };
    const fields = fieldsRef.current;
    let field = fields[name] as Field;

    if (type) {
        field = isRadioOrCheckbox ?
            {
                options: [
              ...unique((field && field.options) || []),
                    {
                        ref,
                        mutationWatcher,
              }
                    as RadioOrCheckboxOption,
            ],
                ref: {
                    type,
                    name
                },
                ...validateOptions,
            } :
            {
                ...fieldRefAndValidationOptions,
                mutationWatcher,
            };
    } else {
        field = fieldRefAndValidationOptions;
    }
    fields[name] = field; // 收集字段的值
    //...
}

// 主动更新字段的值
const setFieldValue = React.useCallback(
    ({
            ref,
            options
        }: Field,
        rawValue:
        |
        FieldValue < TFieldValues >
        |
        UnpackNestedValue < DeepPartial < TFieldValues >>
        |
        undefined |
        null |
        boolean,
    ) => {
        const value =
            isWeb && isHTMLElement(ref) && isNullOrUndefined(rawValue) ?
            '' :
            rawValue;

        if (isRadioInput(ref) && options) {

        } else if (isFileInput(ref) && !isString(value)) {
        } else if (isMultipleSelect(ref)) {
        } else if (isCheckBoxInput(ref) && options) {
        } else {
            ref.value = value; // 非受控组件的方式来更新字段的值
        }
    },
    [],
);
```

#### 2.1.2 使用useState来控制组件的刷新

useRef和非受控组件的优势是不会自动触发组件刷新，但是有些情况下字段值的修改伴随的UI的变化，此时是需要组件进行刷新的。所以react-hook-form中唯一使用了一个**useState()，专门用来刷新组件，这个state本身的值没有任何意义**

```js
const [, render] = React.useState();

const reRender = React.useCallback(
  () => !isUnMount.current && render({}), // 每次调用render传入的都是一个新的对象，所以每次调用都会刷新组件
  [],
);
```

### 2.2 提供了字段的watch功能

由于react-hook-form独特的值管理方式，导致字段的值更新的时候不会自动刷新表单组件。但是某些场景下需要根据某个字段的值变化来更新UI，所以**react-hook-form提供了watch功能来设置需要观察的字段，当这个字段的值发生变化的时候，会刷新组件**。

```js
/**
   * 设置观察字段，返回该字段当前的值。
   * 观察某个字段往往涉及到字段之间的联动效果。
   * 被观察的字段会暂存在 watchFieldsHookRef 或者 watchFieldsRef 中 
   * 会在 isFieldWatched 和 renderWatchedInputs 方法中用到
   * 被观察的字段值更新的时候会刷新组件
   */
const watchInternal = React.useCallback(
    (
        fieldNames ? : string | string[],
        defaultValue ? : unknown,
        watchId ? : string,
    ) => {
        const watchFields = watchId ?
            watchFieldsHookRef.current[watchId] :
            watchFieldsRef.current; // 被watch的字段会记录在这个变量中,两个不同的存储地方，对应下面两种watch的方式
        const combinedDefaultValues = isUndefined(defaultValue) ?
            defaultValuesRef.current :
            defaultValue;
        const fieldValues = getFieldsValues < TFieldValues > (
            fieldsRef,
            unmountFieldsStateRef,
            fieldNames,
        );

        if (isString(fieldNames)) {
            return assignWatchFields < TFieldValues > (
                fieldValues,
                fieldNames,
                watchFields,
                isUndefined(defaultValue) ?
                get(combinedDefaultValues, fieldNames) :
                (defaultValue as UnpackNestedValue < DeepPartial < TFieldValues >> ),
                true,
            );
        }

        if (isArray(fieldNames)) {
            //...
        }

        if (isUndefined(watchId)) {
            isWatchAllRef.current = true;
        }

        return transformToNestObject(
            (!isEmptyObject(fieldValues) && fieldValues) ||
            (combinedDefaultValues as FieldValues),
        );
    },
    [],
);
```

React-hook-form提供了两种watch方式：

1. 字段变化的时候**刷新整个表单组件**，使用useForm()返回的watch方法来观察组件
   
   ```js
   /** 被watch的字段会存在 watchFieldsRef.current中 */
   
   // handleChangeRef.current 方法中有以下代码
   let shouldRender = setDirty(name) || isFieldWatched(name);
   ```

2. 隔离刷新，字段变化的时候只**刷新部分组件**
   
   ```js
   /** 被watch的字段会存在 watchFieldsHookRef.current中 */
   
   // useWatch.ts 中有以下代码
   // 这里多余使用一个useState，就是为了隔离刷新
   const [value, setValue] = React.useState < unknown > (
        isUndefined(defaultValue) ?
        isString(name) ?
        get(defaultValuesRef.current, name) :
        isArray(name) ?
        name.reduce(
            (previous, inputName) => ({
                ...previous,
                 [inputName]: get(defaultValuesRef.current, inputName),
            }), {},
        ) :
        defaultValuesRef.current :
        defaultValue,
    );
   
     React.useEffect(() => {
       const id = (idRef.current = generateId());
       const watchFieldsHookRender = watchFieldsHookRenderRef.current;
       const watchFieldsHook = watchFieldsHookRef.current;
       watchFieldsHook[id] = new Set();
       watchFieldsHookRender[id] = updateWatchValue;
       watchInternal(nameRef.current, defaultValueRef.current, id);
   
       return () => {
         delete watchFieldsHook[id];
         delete watchFieldsHookRender[id];
       };
     }, [
       nameRef,
       updateWatchValue,
       watchFieldsHookRenderRef,
       watchFieldsHookRef,
       watchInternal,
       defaultValueRef,
     ]);
   ```

### 2.3 兼容受控组件

在实际的项目中，可能你已经使用了某组件库，然后可能也想使用react-hook-form。所以react-hook-form使用一种巧妙的方式兼容了受控组件。

```js
// controller.tsx文件中

// 注册字段
// 构造一个假的ref对象用来将字段注册到form层统一管理
// 并且劫持该字段的set和get方法，同步更新到受控字段
const registerField = React.useCallback(() => {
    if (fieldsRef.current[name]) {
      fieldsRef.current[name] = {
        ref: fieldsRef.current[name]!.ref,
        ...rules,
      };
    } else {
      register(
        Object.defineProperty({ name, focus: onFocusRef.current }, VALUE, {
          set(data) {
            setInputStateValue(data);
            valueRef.current = data;
          },
          get() {
            return valueRef.current;
          },
        }),
        rules,
      );
    }
  }, [fieldsRef, rules, name, onFocusRef, register]);

// 修改受控字段的时，使用api同步更新到form层中
const onChange = (...event: any[]) =>
    setValue(name, commonTask(event), {
      shouldValidate: shouldValidate(),
      shouldDirty: true,
    });
```

**本质上是通过高阶组件的方式，将useRef和受控组件联动起来。其中一边值改变的时候，同时需要修改另一边的值。**

### 2.4 进一步的性能优化，不做无意义的刷新

在上面第一点已经讲到，react-hook-form采用useRef和非受控组件的方式，很大程度上减少了组件刷新的次数。但是实际的场景中，可能会根据表单的状态来更新UI，这就意味着表单的状态变化的时候可能也要刷新组件。

首先想到的方案可能是直接为每一个状态建立一个useState()，然后状态改变的时候通过调用状态的set方法来刷新组件。但是这种方式也是相当浪费的，因为**表单某个状态变化往往是由字段的值变化引起的，但是这个状态不一定被外部使用，也就是说这里很有可能做了一次没有意义的刷新。**

React-hook-form内部使用 Proxy 来记录表单的某个状态值是否有被外部组件所使用。如果使用了，就在这个状态变化的时候刷新组件；反之则不会刷新组件。

```js
// 表单的状态
const formState = {
    dirtyFields: dirtyFieldsRef.current,
    isSubmitted: isSubmittedRef.current,
    submitCount: submitCountRef.current,
    touched: touchedFieldsRef.current,
    isDirty: isDirtyRef.current,
    isSubmitting: isSubmittingRef.current,
    isValid: isOnSubmit
      ? isSubmittedRef.current && isEmptyObject(errorsRef.current)
      : isValidRef.current,
  };

// 暴露表单属性的同时劫持访问
// 同时在 readFormStateRef 中记录是否被访问
formState: isProxyEnabled
      ? new Proxy<FormStateProxy<TFieldValues>>(formState, {
          get: (obj, prop: keyof FormStateProxy) => {
            if (prop in obj) {
              readFormStateRef.current[prop] = true;
              return obj[prop];
            }

            return undefined;
          },
        })
      : formState,
```

## 3. React-hook-form的缺点

1. 大量使用useRef()，而且使用的是手动刷新。使代码看起来有点乱，逻辑比较跳跃，不好扩展和维护。增加一个新的功能可能会涉及到多个地方的修改。

## 4. 参考文章

1. [react-hook-form 源码](https://github.com/careyke/react-hook-form) — fork到个人仓库，增加了消息的注释
2. [react-hook-form 官网](https://react-hook-form.com/zh/api/#Controller)
