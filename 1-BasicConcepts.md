## 官方文档学习
### 1. 命名风格
#### 1.1 Class
虚幻的类命名风格，public方法在最前面，带有UCLASS与GENERATED_BODY宏
```cpp
UCLASS()

class EXAMPLEPROJECT_API AExampleActor : public AActor
{
    GENERATED_BODY()
    
public:	
    // Sets default values for this actor's properties
    AExampleActor();

protected:
    
    // Called when the game starts or when spawned
    virtual void BeginPlay() override;
};
```
1. 模板类以T作为前缀
    ```cpp
    template <typename ObjectType>
    class TAttribute
    ```
2. 继承于UObject的以U作为前缀   
    ```cpp
    class UActorComponent
    ```
3. 继承于AActor的以A作为前缀
    ```cpp
    class AActor
    ```
4. 继承于SWidget的以S作为前缀
    ```cpp
    class SCompoundWidget
    ```
5. 抽象接口类以I作为前缀
    ```cpp
    class IAnalyticsProvider
    ```
6. Concept约束以C作为前缀（C++20与Epic实现不一样）
    ```cpp
    struct CStaticClassProvider
    {
        template <typename T>
        auto Requires(UClass*& ClassRef) -> decltype(
            ClassRef = T::StaticClass()
        );
    };
    ```
7. 枚举类以E作为前缀
    ```cpp
    enum class EColorBits
    {
        ECB_Red,
        ECB_Green,
        ECB_Blue
    };
    ```
8. bool变量以小b作为前缀
    ```cpp
    bPendingDestruction
    bHasFadedIn 
    ```

9. 大部分其它类以F作为前缀，除个别子系统外；使用typedef时保留前缀

10. 实例化模板类后可去除T前缀，但保留后一个前缀
    ```cpp
    typedef TArray<FMytype> FArrayOfMyTypes;
    ```
11. 前缀在C#被忽略

12. UHT会使用前缀生成反射信息，因此保持前缀必须正确！

13. 使用In前缀消除模板参数与别名的歧义：
    ```cpp
    template <typename InElementType>
    class TContainer
    {
    public:
        using ElementType = InElementType;
    };
    ```

14. 所有的宏都大写以UE作为前缀并以下划线分割


15. 关于STL的混用问题
    |Idiom|Description|
    |:--:|:--:|
    | \<atomic\> |STL的原子库可以在新代码以及接触到的迁移旧代码使用，epic不会重写原子库|
    |\<type_traits\>|The type traits idiom should be used where there's overlap between a legacy UE trait and a standard trait. Traits are often implemented as compiler intrinsics for correctness, and compilers can have knowledge of the standard traits and select faster compilation paths instead of treating them as plain C++. One concern is that our traits typically have an upper case Value static or Type typedef, whereas standard traits are expected to use value and type. This is an important distinction, as a particular syntax is expected by compositional traits, for example std::conjunction. New traits we add should be written with lowercase value or type to support composition. Existing traits should be updated to support either case.|
    |<initializer_list>|STL是唯一选择|
    |<regex>|正则化库可以使用，但仅限编辑器代码|
    |<limits>|可以使用|
    |<cmath>|可以使用此标头中的所有浮点函数。|
    |<cstring>: memcpy() and memset()|当这有明显的性能优势时，可以分别使用它们来代替FMemory:：Memcpy和FMemory：：Memset。|

16. Const 正确性
    ```cpp
    void SomeMutatingOperation(FThing& OutResult, const TArray<Int32>& InArray)
    {
        // InArray will not be modified here, but OutResult probably will be
    }

    void FThing::SomeNonMutatingOperation() const
    {
        // This code will not modify the FThing it is invoked on
    }

    TArray<FString> StringArray;
    for (const FString& : StringArray)
    {
        // The body of this loop will not modify StringArray
    }
    ```

17. 不要直接使用compiler的c++特性,使用UE的wrap版本

18. 避免bool类型的参数
    ```cpp
    // Old style
    FCup* MakeCupOfTea(FTea* Tea, bool bAddSugar = false, bool bAddMilk = false, bool bAddHoney = false, bool bAddLemon = false);
    FCup* Cup = MakeCupOfTea(Tea, false, true, true);

    // New style
    enum class ETeaFlags
    {
        None,
        Milk  = 0x01,
        Sugar = 0x02,
        Honey = 0x04,
        Lemon = 0x08
    };
    ENUM_CLASS_FLAGS(ETeaFlags)

    FCup* MakeCupOfTea(FTea* Tea, ETeaFlags Flags = ETeaFlags::None);
    FCup* Cup = MakeCupOfTea(Tea, ETeaFlags::Milk | ETeaFlags::Honey);
    ```

19. 避免bool和string的重载，UE的TEXT宏会自动转换
    ```cpp
    void Func(const FString& String);
    void Func(bool bBool);

    Func(TEXT("String")); // Calls the bool overload!
    ```

20. UObject不建议传入引用:
    ```cpp
    // Bad
    void AddActorToList(AActor& Obj);

    // Good
    void AddActorToList(AActor* Obj);
    ```