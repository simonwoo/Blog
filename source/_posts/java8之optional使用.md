title: Java8之Optional使用
date: 2016-1-7 
tags: [Java, Java8] #文章标签，可空，多标签请用格式，注意:后面有个空格
---
## 背景

```
------------------------
|       computer       |
| -------------------- |
| |     soundcard    | |
| | ---------------- | |
| | |     USB      | | |
| | | ------------ | | |
| | | | version  | | | |
| | | ------------ | | |
| | ---------------- | |
| -------------------- |
------------------------
```

让我们以电脑的结构为例，说明Optional的使用。电脑通常情况下包含声卡，声卡里包含USB，每个USB有对应的版本号。
如果想获得声卡中USB的版本号，我们通常使用以下方法：

```java
String version = computer.getSoundcard().getUSB().getVersion();
```

这段代码有潜在的风险。某些情况下，电脑也不一定具备声卡，如Raspberry Pi(树莓派)。所以`getSoundcard()`有可能返回一个null值。`getUSB()`则返回一个null值得USB，这会导致运行期抛出NullPointerException。为了阻止这种情况的发生，我们经常会进行null值检查：

```java
String version = UNKNOWN;
if(computer != null){
  Soundcard soundcard = computer.getSoundcard();
  if(soundcard != null){
    USB usb = soundcard.getUSB();
    if(usb != null){
      version = usb.getVersion();
    }
  }
}
```

由于多重嵌套进行null坚持，这段代码非常丑陋。让我们看看Optional是怎么完成相同的功能：

```java
String version = computer.flatMap(Computer::getSoundcard)
                          .flatMap(Soundcard::getUSB)
                          .map(USB::getVersion)
                          .orElse("UNKNOWN");
```

## Optional使用
### 创建Optional对象

```java
Optional<Soundcard> sc = Optional.empty(); //an empty Optional object

SoundCard soundcard = new Soundcard();
Optional<Soundcard> sc = Optional.of(soundcard); //an non-null Optional object

Optional<Soundcard> sc = Optional.ofNullable(soundcard); // an Optional object that may hold a null value
```

### 操作对象

```java
Optional<Soundcard> soundcard = ...;
soundcard.ifPresent(System.out::println);

Soundcard soundcard = maybeSoundcard.orElse(new Soundcard("defaut"));

Soundcard soundcard = maybeSoundCard.orElseThrow(IllegalStateException::new);

/**
* The filter method takes a predicate as an argument.
* If a value is present in the Optional object and it matches the predicate, the filter method returns that value;
* otherwise, it returns an empty Optional object.
**/
Optional<USB> maybeUSB = ...;
maybeUSB.filter(usb -> "3.0".equals(usb.getVersion())
                    .ifPresent(() -> System.out.println("ok"));

// extracting and transforming values using map, nothing happens if Optional object is empty                  
Optional<USB> usb = maybeSoundcard.map(Soundcard::getUSB);      

/**
* Optional also supports a flatMap method.
* Its purpose is to apply the transformation function on the value of an Optional (just like the map operation does) and then flatten the * resulting two-level Optional into a single one
**/
String version = computer.flatMap(Computer::getSoundcard)
                   .flatMap(Soundcard::getUSB)
                   .map(USB::getVersion)
                   .orElse("UNKNOWN");
```
