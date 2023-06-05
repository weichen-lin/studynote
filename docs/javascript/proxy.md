---
sidebar_position: 1
---

# Proxy

> 非原創內容，資料來源請見 `reference`

![JS proxy](https://image.fundebug.com/2019-07-27.png 'JS proxy')

Proxy 物件可以在目標對象前面再加一層攔截，代理物件的行為。

### Proxy 可以代理的行為

- get
- set
- has
- apply
- construct
- ownKeys
- deleteProperty
- defineProperty
- isExtensible
- preventExtensions
- getPrototypeOf
- setPrototypeOf
- getOwnPropertyDescriptor

---

### Example 1 : 加上默認值

JS 的物件在未被賦值的時候，都會是 `undefined`，沒有類型安全的零值特性，用 Proxy 可以得出一個有默認安全零值的物件。

```typescript
const withZeroValue = (target, zeroValue) => {
  return new Proxy(target, {
    get: (obj, prop) => (prop in obj ? obj[prop] : zeroValue),
  })
}

const ex1 = {
  a: 1,
  b: 2,
}
console.log(ex1.a, ex1.b, ex1.c)
// 1, 2, undefined

const ex2 = withZeroValue(
  {
    a: 1,
    b: 2,
  },
  0
)
console.log(ex2.a, ex2.b, ex2.c)
// 1, 2, 0
```

---

### Example 2 : Array 負數索引

Python 可以簡單透過 `arr[-1]`拿到最後一個元素，但是 JS 會比較冗長且重複

用 Proxy 實踐 array 負數索引

```typescript
const negativeArray = (els) =>
  new Proxy(els, {
    get: (target, propKey, receiver) =>
      Reflect.get(
        target,
        propKey < 0 ? String(target.length + propKey) : propKey,
        receiver
      ),
  })

const a = negativeArray([1, 2, 3, 4, 5, 6])
a[-1]
// 6
```

---

### Example 3 : 私有屬性

JS 沒有私有屬性(可以透過 `Symbol`來實作類似的)
可以透過 `Proxy`實現

```typescript
const Hide = (target, prefix = '_') => {
  return new Proxy(target, {
    has: (obj, prop) => !prop.starsWith(prefix) && prop in obj,
    ownKeys: (obj) =>
      Reflect.ownKeys(obj).filter(
        (prop) => typeof prop !== 'string' || !prop.startsWith(prefix)
      ),
    get: (obj, prop, rec) => (prop in rec ? obj[prop] : undefined),
  })
}
```

---

### Example 4 : Cache

```typescript
const ephemeral = (target, ttl = 60) => {
  const CREATED_AT = Date.now()
  const isExpired = () => Date.now() - CREATED_AT > ttl * 1000

  return new Proxy(target, {
    get: (obj, prop) => (isExpired() ? undefined : Reflect.get(obj, prop)),
  })
}

let bankAccount = ephemeral(
  {
    balance: 14.93,
  },
  10
)

console.log(bankAccount.balance) // 14.93

setTimeout(() => {
  console.log(bankAccount.balance) // undefined
}, 10 * 1000)
```

---

### Example 5 : Create Enum

```typescript
const NOPE = () => {
  throw new Error("Can't modify read-only view")
}

const NOPE_HANDLER = {
  set: NOPE,
  defineProperty: NOPE,
  deleteProperty: NOPE,
  preventExtensions: NOPE,
  setPrototypeOf: NOPE,
}

const readOnlyView = (target) => new Proxy(target, NOPE_HANDLER)

const createEnum = (target) =>
  readOnlyView(
    new Proxy(target, {
      get: (obj, prop) => {
        if (prop in obj) {
          return Reflect.get(obj, prop)
        }
        throw new ReferenceError(`Unknown prop "${prop}"`)
      },
    })
  )

let SHIRT_SIZES = createEnum({
  S: 10,
  M: 15,
  L: 20,
})

SHIRT_SIZES.S // 10
SHIRT_SIZES.S = 15

// Uncaught Error: Can't modify read-only view

SHIRT_SIZES.XL

// Uncaught ReferenceError: Unknown prop "XL"
```

---

### Example 6 : Handle Cookie

```typescript
const getCookieObject = () => {
  const cookies = document.cookie.split(';').reduce(
    (cks, ck) => ({
      [ck.substr(0, ck.indexOf('=')).trim()]: ck.substr(ck.indexOf('=') + 1),
      ...cks,
    }),
    {}
  )
  const setCookie = (name, val) => (document.cookie = `${name}=${val}`)
  const deleteCookie = (name) =>
    (document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:01 GMT;`)

  return new Proxy(cookies, {
    set: (obj, prop, val) => (
      setCookie(prop, val), Reflect.set(obj, prop, val)
    ),
    deleteProperty: (obj, prop) => (
      deleteCookie(prop), Reflect.deleteProperty(obj, prop)
    ),
  })
}

let docCookies = getCookieObject()

docCookies.has_recent_activity // "1"
docCookies.has_recent_activity = '2' // "2"
delete docCookies2['has_recent_activity'] // true
```

### 不可代理的物件

如果你的物件擁有 `configurable: false` 與 `writable: false` 的屬性，那該物件就無法被 proxy 代理：

```typescript
const target = Object.defineProperties(
  {},
  {
    FooBar: {
      writable: false,
      configurable: false,
    },
  }
)
const handler = {
  get(target, propKey) {
    return '???'
  },
}
const proxy = new Proxy(target, handler)
proxy.FooBar
// Uncaught TypeError: 'get' on proxy: property 'FooBar' is a read-only and non-configurable data property on the proxy target but the proxy did not return its actual value (expected 'undefined' but got '???')
```

<br/>
<br/>

> **Reference**
>
> 1. https://blog.fundebug.com/2019/07/27/javascript-es6-how-to-use-proxy/
> 2. https://blog.techbridge.cc/2018/05/27/js-proxy-reflect/
> 3. https://blog.techbridge.cc/2018/05/27/js-proxy-reflect/
> 4. https://javascript.info/proxy
