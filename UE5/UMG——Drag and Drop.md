### HandCard(DraggableWidget)
Uproperty DragOffset

Uproperty bMoveToDropPosition

FEventReply:Allows users to handle events and return information to the underlying UI layer.
```Cpp
FEventReply
(
	bool IsHandled
)	
```

```Cpp
FReply UdraggableWidget::NationOnMouseButtonDown(const Fgeometry& InGeometry, const FPointerEvent& InMouseEvent)
```
首先获得当前Geometry的局部坐标作为偏移量
```cpp
this->DragOffset == InGeometry.AbsoluteToLocal(InMouseEvent.GetScreenSpacePosition());
```
判断当前事件是否是鼠标左键点击事件（InMouseEvent.IsTouchEvent()是触屏模式下的）
然后选择当前拖拽的widget()(this->GetCachedWidget()获取的就是当前widget嘛？)
如果当前widget是有效的，则进行拖拽（DetectDrag()函数）
最后通过Reply.NativeReply来进行拖拽后的信息传递。
```Cpp
if (InMouseEvent.GetEffectingButton() == Ekeys::LeftMouseButton || InMouseEvent.IsTouchEvent)
{
	FEventReply Reply;
	Reply.NativeReply = Freply::Handled();
	TsharedPtr<SWidget> SlateWidgetDetectingDrag = this ->GetCachedWidget();
	if (SlateWidgetDetectingDrag.IsValid())
	{
		Reply.NativeReply = Reply.NatinveReply.DetecDrag(SlateWidgetDetectingDrag.ToShareRef(), EKeys::LeftMouseButton);
		return Reply.NativeReply;
	}
	return Reply.NativeReply;
}
return Super::NativeOnMouseButtonDown(InGeometry, InMouseEvent);
```

OnDragDetect功能：决定玩家实际在屏幕上拖动widget发生的状况
拖拽后，如果之前有正在被拖拽的widget，则把他删除
如果有填写DragWidgetClass的话，则创建填写的DragWidgetClass类的widget（没有的话就创建当前widget类的）
```cpp
if (DragWidget != nullptr)
{
	DragWidget->RemoveFromParent();
	DragWidget = nullptr;
}
if (DragWidgetClass != nullptr)
{
	DragWidget = CreateWidget<UDragWidget>(this->GetOwningPlayer(), DragWidgetClass);
	DragWidget->SetDragginWidget(this);
}
```

然后创建DragDropOperation
将MouseDown时作为作为枢轴(Pivot)
设置拖拽的组件（DragWidget)作为默认可视化拖拽组件（DragDropOperation->DefaultDragVisual)
如果没有的拖拽组件的话将this作为拖拽组件
然后对基本属性进行一些配置
```cpp
UWidgetDragDropOperation* DragDropOperation = NewObject<UWidgetDragDropOperation();
DragDropOperation->Pivot = EDragPivot::MouseDown;
if (DragWidget != nullptr)
{
	DragDropOperation->DefaultDragVisua = DragWidget;
}
else
{  
   DragDropOperation->DefaultDragVisual = this;  
}  
DragDropOperation->DraggingWidget = this;  
DragDropOperation->DragOffset = this->DragOffset;  
DragDropOperation->WidgetOriginalAbsolutePosition = GetCachedGeometry().GetAbsolutePosition();  
OutOperation = DragDropOperation;
```

### CombatWidget
NativeOnDrop
```cpp
bool UCombatWidget::NativeOnDrop(const FGeometry& InGeometry, const FDragDropEvent& InDragDropEvent, UDragDropOperation* InOperation)
```
首先获得鼠标点击的位置，得到DragDropOperation中传入的DraggingWidget
使DragableWidget是否可视参数中确认可视
如果bMoveToDropPosition为True：
	则将原本的widget删除（RemoveFromParent),然后将其在重新加入到拖拽后的位置（拖拽的偏移量减去鼠标的偏移量）
为False：
	放置Widget到原本的位置（OriginalPosition)

```cpp
if (UWidgetDragDropOperation* DragDropOperation = Cast<UWidgetDragDropOperation>(InOperation))  
{  
   FVector2D ScreenSpacePosition = InDragDropEvent.GetScreenSpacePosition();  
   if (UDraggableWidget* DraggableWidget = Cast<UDraggableWidget>(DragDropOperation->DraggingWidget))  
   {  
      DraggableWidget->SetVisibility(ESlateVisibility::Visible);  
      if (DraggableWidget->bMoveToDropPosition)  
      {         DraggableWidget->RemoveFromParent();  
         const FVector2D DragOffset = InGeometry.AbsoluteToLocal(ScreenSpacePosition);  
         DraggableWidget->AddToViewport();  
         DraggableWidget->SetPositionInViewport(DragOffset - DragDropOperation->DragOffset, false);  
      }      else  
      {  
         const FVector2D OriginalPosition = InGeometry.AbsoluteToLocal(DragDropOperation->WidgetOriginalAbsolutePosition);  
         DraggableWidget->SetPositionInViewport(OriginalPosition);  
      }
   }
```

使用流程
以`DraggableWidget`作为基类，创建新的widget，然后再