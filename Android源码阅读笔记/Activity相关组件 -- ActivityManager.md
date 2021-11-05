# Activity相关组件 -- ActivityManager

- 这个类用来提供和Activity、Service、进程有关的信息，并且可以和他们进行交互
- 这个类中大多数方法是用来debug的，大多数APP开发者不需要使用此类

## 代码分析

### 主要方法

- processStateAmToProto(int amInt)

  ```java
  public static final int processStateAmToProto(int amInt) {
      switch (amInt) {//根据传入amInt的值获取AppProtoEnums的值
          case PROCESS_STATE_UNKNOWN:
              return AppProtoEnums.PROCESS_STATE_UNKNOWN;
          case PROCESS_STATE_PERSISTENT:
              return AppProtoEnums.PROCESS_STATE_PERSISTENT;
              ...;
          default:
              return AppProtoEnums.PROCESS_STATE_UNKNOWN_TO_PROTO;
      }
  }
  ```

- 

