- 异常
  - 异常实现是通过异常表，从第x1行到y1行出现异常 跳转至z1行执行，然后**继续顺序执行代码**——因为异常跳过的代码不会在异常处理完就返回接着执行
  - 除了return语句 try 和 catch块执行retrun时，先将返回值写入栈顶等待返回，之后需要执行finally块，之后返回try或catch块的return语句处，直接将栈顶元素return
    - 比如try块return a将此时a的值写入栈顶，之后执行finally块改变a的值，但是函数栈顶元素值不会改变
  
- java没有无符号数

- system.arraycopy

- 拆箱装箱——Integer为例

  - 拆箱就是调用Integer.intValue

    - 返回Integer包装的int类型——在实例化new Integer(x) 时传入

  - 装箱就是Integer.valueOf()

    - 是一个静态工厂的实现

    - 

    - ```java
          public static Integer valueOf(int i) {
            // 判断是否在缓存中，是则从缓存获取
            // 否则new
              if (i >= IntegerCache.low && i <= IntegerCache.high)
                  return IntegerCache.cache[i + (-IntegerCache.low)];
              return new Integer(i);
          }
      
      // 缓存下界是-128 上界可以通过jvm参数指定
      private static class IntegerCache {
              static final int low = -128;
              static final int high;
              static final Integer cache[];
      
              static {
                  // high value may be configured by property
                  int h = 127;
                  String integerCacheHighPropValue =
                      sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
                  if (integerCacheHighPropValue != null) {
                      try {
                          int i = parseInt(integerCacheHighPropValue);
                          i = Math.max(i, 127);
                          // Maximum array size is Integer.MAX_VALUE
                          h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                      } catch( NumberFormatException nfe) {
                          // If the property cannot be parsed into an int, ignore it.
                      }
                  }
                  high = h;
      
                  cache = new Integer[(high - low) + 1];
                  int j = low;
                  for(int k = 0; k < cache.length; k++)
                      cache[k] = new Integer(j++);
      
                  // range [-128, 127] must be interned (JLS7 5.1.7)
                  assert IntegerCache.high >= 127;
              }
      
              private IntegerCache() {}
          }
      ```

  - int -128~127

  - byte 全部—— -128~127

  - boolean 缓存 true和false

  - char 0-127

    - 包括 大小写字母 和数字
    - 还有些常用符号

  - short -128~127

  - long -128~127

  - float double无缓存

- string

  - new String(xx) 会实例化新的字符串 与 xx不等
  - 字符串缓存 **所有的字面量和字符串常量表达式会存储到字符串池**
    - s1 = "a" + "b" s2 = "ab" s1==s2
  - 字符串可不变性
    - s1 = s2 = "123" 修改s2的值，s1不变 s2指向新的字符串
    - trim 根据是否需要切割字符串判断是否需要new 新字符串
      - 假如不需要切割 返回原字符串
    - 运行时计算 不会使用常量池
  - intern
    - 假如当前字符串池中有与当前string equals的字符，那么直接返回池中的string
    - 否则，将当前string存到池中，然后返回一个指向当前字符串的引用(**和原来的不同**)
    - s1.equals(s2) 那么 s1.intern() == s2.intern()
  - 字符串常量池

  