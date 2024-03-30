一些经典的设计原则，其中包括SOLID、KISS、YAGNI、DRY、LOD等
SOLID原则并非单纯的一个原则，而是由五个设计原则组成，它们分别是单一职责原则、开闭原则、里氏替换原则、接口隔离原则和依赖反转原则，依次对应SOLID中的S、O、L、I、D这五个英文字母。
# 1、单一职责原则
单一职责原则的英文是Single Responsibility Principle，缩写为 SRP。这个原则的英文描述是这样的： A class or module should have a single responsibility。如果把它翻译成中文，那就是：一个类或者模块只负责完成一个职责或者功能。  
单一职责原则的定义描述非常简单，也不难理解。一个类只负责完成一个职责或者功能。也就是说，不要设计大而全的类，要设计粒度小、功能单一的类。换个角度来讲就是，一个类包含了两个或者两个以上业务不相干的功能，就可以说它职责不够单一，应该将它拆分成多个功能更加单一、粒度更细的类。
## 1.1 如何判断类的职责是否足够单一？  
- 类中的代码行数、函数或属性过多，会影响代码的可读性和可维护性，我们就需要考虑对类进行拆分
- 类依赖的其他类过多，或者依赖类的其它类过多，不符合高内聚、低耦合的设计思想，我们就需要考虑对类进行拆分
- 私有方法过多，我们就要考虑能否将私有方法独立到新的类中，设置为Public方法，供更多的类使用，从而提高代码的复用性
- 比较难给类起一个合适名字，很难用一个业务名词概括，或者只能用一些笼统的Manager、Context之类的词语来命名，这就说明类的职责定义的可能不够清晰
- 类中大量方法都是集中操作类的某几个属性，就可以考虑将这几个属性和对应的方法拆分出来

## 1.2 类的职责是否设计的越单一越好？
单一职责原则通过避免设计大而全的类，避免将不相关的功能偶合在一起，来提高类的内聚性。同时类职责单一，类依赖的和被依赖的其它类也会变少，减少了代码的耦合性，以此来实现代码的高内聚、低耦合。但是如果拆分的过细，实际上会适得其反，反倒会降低内聚性，也会影响代码的可维护性。

DMSerialization 类实现了简单的序列化和反序列化功能
```
@interface DMSerialization : NSObject

- (NSString *)serialize:(NSDictionary *)dict;

- (NSDictionary *)deSerialize:(NSString *)str;

@end


@implementation DMSerialization

- (NSString *)serialize:(NSDictionary *)dict
{
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:dict options:0 error:nil];
    NSString *jsonStr = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
    
    return jsonStr;
}

- (NSDictionary *)deSerialize:(NSString *)str
{
    NSData *jsonData = [str dataUsingEncoding:NSUTF8StringEncoding];
    NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:jsonData options:0 error:NULL];
    
    return dict;
}

@end
```
如果想让类的职责更加单一，需要对DMSerialization类进一步拆分，拆分成一个只负责序列化工作的DMSerializer类和另一个只负责反序列化工作的DMDeserializer类

DMSerializer类
```
@interface DMSerializer : NSObject

- (NSString *)serialize:(NSDictionary *)dict;

@end

@implementation DMSerializer

- (NSString *)serialize:(NSDictionary *)dict
{
    NSData *jsonData = [NSJSONSerialization dataWithJSONObject:dict options:0 error:nil];
    NSString *jsonStr = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
    
    return jsonStr;
}

@end
```
DMDeserializer 类
```
@interface DMDeserializer : NSObject

- (NSDictionary *)deSerialize:(NSString *)str;

@end

@implementation DMDeserializer

- (NSDictionary *)deSerialize:(NSString *)str
{
    NSData *jsonData = [str dataUsingEncoding:NSUTF8StringEncoding];
    NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:jsonData options:0 error:NULL];
    
    return dict;
}

@end
```

虽然经过拆分之后，DMSerializer类和DMDeserializer类职责更加单一了，但随之也带来了新的问题。比如序列化方式从JSON改为了XML，那DMSerializer类和DMDeserializer类都需要做相应的修改，代码的内聚性显然没有原来的DMSerialization高了。而且如果仅对DMSerializer类做了协议修改，而忘了修改DMDeserializer类的代码，那就会导致序列化、反序列化不匹配，程序运行出错，也就是说，拆分之后，代码的可维护性变差了。  