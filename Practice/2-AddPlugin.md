## 为虚幻引擎添加Plugin
### 1. 创建Scripted Asset Action
在创建的空Plugin中，创建一个类继承自`QuickAssetAction`，当为其中的成员函数设置`UFUNCTION(CallInEditor)`，即可在编辑器中使用。

> 此处有一个bug，无法在蓝图中查看到Scripted Asset Action的按钮，暂不知原因，但若该蓝图类继承自`QuickAssetAction`，则可以右键看到该按钮


```cpp
// QuickAssetAction.h
UCLASS()
class TESTPLUG_API UQuickAssetAction : public UAssetActionUtility
{
	GENERATED_BODY()
public:
	UFUNCTION(CallInEditor)
	void TestFunc();
};

// QuickAssetAction.cpp
void UQuickAssetAction::TestFunc()
{
	if(GEngine)
	{
		GEngine->AddOnScreenDebugMessage(-1,8.f,FColor::Cyan,TEXT("Plugin Working"));
	}
}

```
![alt text](image-plugin002.png)
![alt text](image-plugin001.png)

基于此，可以利用`EditorUtilityLibrary`和`EditorAssetLibrary`提供的增删改查（Get/Set）方法，根据实际的需要，实现一些资产复制，重新命名等快捷的功能。
例如下面，简单实现一个复制多个蓝图资产的功能：
```cpp
void UQuickAssetAction::DuplicateAssets(int32 NumOfDuplicates)
{
	if (NumOfDuplicates < 0)
	{
		TestPluginPrint(TEXT("Please Input Valid NumOfDuplicates"), FColor::Red);
                return ;
	}

	uint16_t Counter = 0;
	
	TArray<UObject*> SelectedAssets = UEditorUtilityLibrary::GetSelectedAssets();
	for(const FAssetData& SelectedAssetData:SelectedAssets)
	{
		for(int32 i = 0; i < NumOfDuplicates; i++)
		{
			const FString SourceAssetPath = SelectedAssetData.ObjectPath.ToString();
			const FString NewDuplicatedAssetName = SelectedAssetData.AssetName.ToString() + TEXT("_") + FString::FromInt(i+1);
			const FString NewPathName = FPaths::Combine(SelectedAssetData.PackagePath.ToString(), NewDuplicatedAssetName);

			if(UEditorAssetLibrary::DuplicateAsset(SourceAssetPath, NewPathName))
			{	
				UEditorAssetLibrary::SaveAsset(NewPathName,false);
				++Counter;
			}
		}
	}

	TestPluginPrint(TEXT("Successfully duplicated " + FString::FromInt(Counter) + " files"), FColor::Green);
}

```