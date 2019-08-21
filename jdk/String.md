### 理解 ###
+ String底层实现为char[]数组，所有对字符串修改的操作都会新生成一个String对象（replace/replaceAll/substring/concat/trim/toLowerCase/toUpperCase）
+ String实现了hashCode()和equals()方法
+ indexOf方法：循环char[]数组，==时返回索引位置
+ startWith方法：从指定位置（默认为0）开始循环比较，有==为false的直接返回false
+ substring方法：底层调用了数组拷贝（Arrays.copyOfRange），从char[]数组指定范围copy之后，重新生成一个String对象
+ replace方法：先循环char[]确认是否有被替换元素，没有直接返回。之后再次缓存char[]数组，一个个判断是否相等，相等用新字符，不等用旧字符
+ equals方法：两字符串的char数组，循环比较，不等直接返回false(前提是这两char[]长度相等，不然直接返回false)