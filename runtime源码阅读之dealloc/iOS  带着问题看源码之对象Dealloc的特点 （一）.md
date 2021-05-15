iOS  带着问题看源码之对象Dealloc的特点 （一）

- 在调用对象的dealloc时，它的实例变量（ivars）还没有被销毁，根类调用dealloc方法时，才会销毁
- 父类的dealloc会被自动调用
- 在dealloc时，会将关联属性清除掉
- 在dealloc时，清除弱引用表
