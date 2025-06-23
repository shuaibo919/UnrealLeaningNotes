
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
    |              Idiom               |                                                                                                                                                                                                                                                                                                                                                                       Description                                                                                                                                                                                                                                                                                                                                                                       |
    | :------------------------------: | :-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
    |            \<atomic\>            |                                                                                                                                                                                                                                                                                                                                          STL的原子库可以在新代码以及接触到的迁移旧代码使用，epic不会重写原子库                                                                                                                                                                                                                                                                                                                                          |
    |         \<type_traits\>          | The type traits idiom should be used where there's overlap between a legacy UE trait and a standard trait. Traits are often implemented as compiler intrinsics for correctness, and compilers can have knowledge of the standard traits and select faster compilation paths instead of treating them as plain C++. One concern is that our traits typically have an upper case Value static or Type typedef, whereas standard traits are expected to use value and type. This is an important distinction, as a particular syntax is expected by compositional traits, for example std::conjunction. New traits we add should be written with lowercase value or type to support composition. Existing traits should be updated to support either case. |
    |        <initializer_list>        |                                                                                                                                                                                                                                                                                                                                                                      STL是唯一选择                                                                                                                                                                                                                                                                                                                                                                      |
    |             <regex>              |                                                                                                                                                                                                                                                                                                                                                           正则化库可以使用，但仅限编辑器代码                                                                                                                                                                                                                                                                                                                                                            |
    |             <limits>             |                                                                                                                                                                                                                                                                                                                                                                        可以使用                                                                                                                                                                                                                                                                                                                                                                         |
    |             <cmath>              |                                                                                                                                                                                                                                                                                                                                                            可以使用此标头中的所有浮点函数。                                                                                                                                                                                                                                                                                                                                                             |
    | <cstring>: memcpy() and memset() |                                                                                                                                                                                                                                                                                                                                   当这有明显的性能优势时，可以分别使用它们来代替FMemory:：Memcpy和FMemory：：Memset。                                                                                                                                                                                                                                                                                                                                   |

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


### 2. UE中的关卡，世界和上下文
#### 2.1 Level

ULevel是UE的关卡，是场景中物体的集合，存储着一系列Actor，包含可见物体（如网格体、灯光、特效等）以及不可见物体（如体积、蓝图、关卡配置、导航数据等）

#### 2.2 World

UWorld是ULevel的容器，它才真正地代表着一个场景，因为ULevel必须放置到UWorld才能显示出其内容。每个UWorld实例必须包含一个主关卡（Persistent Level），还可能包含若干个流式关卡（Streaming Level，可选，非必需，可按需动态加载和卸载）。除了关卡信息，UWorld还保存着Scene、GameInstance、AISystem、FXSystem、NavigationSystem、PhysicScene、TimerManager等等信息。它有以下几种类型:

```cpp
// Engine\Source\Runtime\Engine\Classes\Engine\EngineTypes.h

namespace EWorldType
{
	enum Type
	{
		/** An untyped world, in most cases this will be the vestigial worlds of streamed in sub-levels */
		None,
		/** The game world */
		Game,
		/** A world being edited in the editor */
		Editor,
		/** A Play In Editor world */
		PIE,
		/** A preview world for an editor tool */
		EditorPreview,
		/** A preview world for a game */
		GamePreview,
		/** A minimal RPC world for a game */
		GameRPC,
		/** An editor world that was loaded but not currently being edited in the level editor */
		Inactive
	};
}
```
#### 2.3 Context
FWorldContext是引擎层面处理Level的设备上下文，方便UEngine管理和记录World关联的信息。用于内部类，不应该被逻辑层直接操作。它存储的数据有World类型、ContextHandle、GameInstance、GameViewport等等信息。

#### 2.4 UEngine
UEngine控制和掌管着很多内部系统及资源，下派生出UGameEngine和UEditorEngine。它是一个单例的全局变量：
```cpp
// Engine/Source/Runtime/Engine/Classes/Engine/Engine.h
/** Global engine pointer. Can be 0 so don't use without checking. */
extern ENGINE_API class UEngine*			GEngine;
```

它是在程序启动之初在FEngineLoop::PreInitPostStartupScreen被创建并赋值的：


#### 2.5 启动
在不同的操作系统，有着不同的入口，比如Windows的程序入口是WinMain，而Linux是Main。但所有的入口最后都会调用GuardedMain函数，它是UE的入口函数。
这段逻辑主要有4个步骤：引擎预初始化（EnginePreInit）、引擎初始化（EngineInit）、引擎帧更新（EngineTick）、引擎退出（EngineExit）
```cpp
// Engine\Source\Runtime\Launch\Private\Launch.cpp

int32 GuardedMain( const TCHAR* CmdLine )
{
    (......)

    // 保证能够调用EngineExit
    struct EngineLoopCleanupGuard 
    { 
        ~EngineLoopCleanupGuard()
        {
            EngineExit();
        }
    } CleanupGuard;

    (......)

    // 引擎预初始化
    int32 ErrorLevel = EnginePreInit( CmdLine );
    if ( ErrorLevel != 0 || IsEngineExitRequested() )
    {
        return ErrorLevel;
    }

    {
        (......)

#if WITH_EDITOR
        if (GIsEditor)
        {
            // 编辑器初始化
            ErrorLevel = EditorInit(GEngineLoop);
        }
        else
#endif
        {
            // 引擎(非编辑器)初始化
            ErrorLevel = EngineInit();
        }
    }

    (......)

    while( !IsEngineExitRequested() )
    {
        // 引擎帧更新
        EngineTick();
    }

#if WITH_EDITOR
    if( GIsEditor )
    {
        // 编辑器退出
        EditorExit();
    }
#endif
    return ErrorLevel;
}
```

##### 2.5.1 引擎预初始化
UE引擎预初始化主要是在启动页面期间做的很多初始化和基础核心相关模块的事情，预初始化阶段会初始化随机种子，加载CoreUObject模块，启动FTaskGraphInterface模块并将当前游戏线程附加进去，之后加载UE的部分基础核心模块（Engine、Renderer、SlateRHIRenderer、Landscape、TextureCompressor等），由LoadPreInitModules完成。
随后处理的是配置Log、加载进度信息、内存分配器的TLS（线程局部范围）缓存、设置部分全局状态、处理工作目录、初始化部分基础核心模块（FModuleManager、IFileManager、FPlatformFileManager等）。还有比较重要的一点：处理游戏线程，将当前执行WinMain的线程设置成游戏线程（主线程）并记录线程ID。