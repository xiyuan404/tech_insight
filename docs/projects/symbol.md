- Symbol() ：内部会建立一个独特的 id（unique id） ，两个相同 key 的 Symbol 是不同的，且无法透过 Symbol.keyFor() 找到。
- Symbol.for() ：一样都会生成新的 Symbol，但两个相同 key 的 Symbol 会是相同的（reused id），且可以透过 Symbol.keyFor() 找到。
- Symbol.keyFor() 方法返回一个已登记的 Symbol 类型值的 key`



prefix and postfix

origin, target


bannerId \u+001
id ❌