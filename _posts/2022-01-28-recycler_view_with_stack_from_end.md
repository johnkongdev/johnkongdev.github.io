---
title: StackFromEnd 가 의도와 다르게 동작했던 일 
date: 2022-01-28 22:00:00 +0900
categories: [Android, View]
tags: [android, recyclerview, stackfromend, constraintlayout]
img_path: /img/3
---


아래 조건을 만족하는 화면을 개발 중이었습니다. (우리가 흔히 쓰는 메신저 같은)

# 요구사항
#### 0. 목록에는 메시지들이 있습니다.
#### 1. 첫 진입시 가장 아래있는 (최근) 메시지부터 보여준다. (스크롤 최하단)
#### 2. RecyclerView 에 아이템을 추가하면 아래로 쌓인다.

----

여기서 1번 요구사항을 만족하기 위해 우리는 보통 `setStackFromEnd(true);` 를 사용합니다.

```java
private void initRecyclerView() {
    final LinearLayoutManager linearLayoutManager = new LinearLayoutManager(context);
    linearLayoutManager.setStackFromEnd(true);
    binding.messageList.setLayoutManager(linearLayoutManager);
    binding.messageList.setAdapter(adapter);
}
```

LinearLayoutManager 의 StackFromEnd 를 설정해주면 ~~아주 간단히 해결됩니다.~~

될 줄 알았는데, 한가지 이슈가 있었습니다.

# 문제상황
## 아래와 같은 layout 에서 stackFromEnd 는 올바르게 동작할까요?

 (이번 이슈와 무관한 부분은 과감히 뺐습니다)

```xml
<androidx.constraintlayout.widget.ConstraintLayout>

    <View
        android:id="@+id/channel_app_bar"
        android:layout_width="match_parent"
        android:layout_height="56dp"
        android:background="#abab99"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"/>

    <View
        android:id="@+id/channel_app_bar_divider"
        android:layout_width="match_parent"
        android:layout_height="0.5dp"
        android:background="@color/channel_app_bar_divider_color"
        app:layout_constraintTop_toBottomOf="@id/channel_app_bar"/>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/message_list"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        app:layout_constraintTop_toBottomOf="@id/channel_app_bar_divider"
        app:layout_constraintBottom_toTopOf="@id/view_chat_input"/>

    <View
        android:id="@+id/view_chat_input"
        android:layout_width="match_parent"
        android:layout_height="90dp"
        android:background="#99abab"
        app:layout_constraintBottom_toTopOf="@id/bottom_container"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/message_list"/>

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/bottom_container"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="#ab99ab"
        app:layout_constrainedHeight="true"
        app:layout_constraintHeight_max="200dp"
        app:layout_constraintHeight_percent="0.5"
        app:layout_constraintTop_toBottomOf="@id/view_chat_input"
        app:layout_constraintVertical_bias="1"
        app:layout_constraintBottom_toBottomOf="parent"
        android:visibility="gone"/>


</androidx.constraintlayout.widget.ConstraintLayout>
```

상단에는 AppBar 와 얇은 경계선이 있고,

하단에는 FragmentContainerView 가 Bottom 에서부터 올라와서

그 위로 InputView 가 있습니다. 

(TMI지만 FragmentContainerView 는 이모티콘 피커, InputText 는 텍스트 입력창이라고 보면 됩니다)

그리고 RecyclerView 는 상단과 하단의 나머지 공간을 차지합니다.

Android Studio 의 Design 탭에서 랜더링된 이미지는 아래와 같습니다.

각 영역이 잘 연결되어있어 보이고 특별한 문제는 보이지 않습니다.

![3_1](3_1.png)

# 잘 될까요?
잘 되지만, 이상한 점이 있습니다.

원하던대로 리스트의 하단부터 아이템을 쌓아올려서 스크롤이 하단에 가있습니다.

한가지 문제는 *스크롤이 최하단에 가있지 않는 이슈*가 있습니다.

스크롤을 한번 위로 튕겨줘야만 최하단 아이템이 보였습니다.

분명 아래서부터 쌓아 올린거 같긴한데 뭘까요?!

# 왜?!!
*아직 이유는 못찾았습니다.*

RecyclerView 의 높이가 결정되기 전에 아이템이 쌓이면서 뭔가 영향을 줬나 싶었는데,

화면이 idle 상태에서 RecyclerView 에 데이터가 바인딩될 때도 여전히 문제가 있습니다.

왜인지 언젠가 밝혀내면, 아래에 추가해두겠습니다.

다행히 수정하는 방법은 찾았습니다.

# 해결 
RecyclerView 의 width 를 수정하면 됩니다.

## As-is
```xml
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/message_list"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    app:layout_constraintTop_toBottomOf="@id/channel_app_bar_divider"
    app:layout_constraintBottom_toTopOf="@id/view_chat_input"/>
```

## To-be
```xml
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/message_list"
    android:layout_width="0dp"
    android:layout_height="0dp"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintTop_toBottomOf="@id/channel_app_bar_divider"
    app:layout_constraintBottom_toTopOf="@id/view_chat_input"/>
```

`match_parent` -> `0dp` 로 수정했고,
`start & end to parent` 제약조건을 추가했습니다.

조금 황당합니다.

저는 보통 width 제약이 불필요해보일 땐, match_parent 를 쓰곤 했습니다.

그런데, 0dp 로 Left(Start), Right(End) 제약을 추가해줘야만 

내부적으로 계산이 되는 부분이 있나봅니다.

예상컨데, 제약조건이 불분명해서 생긴 문제였던거 같습니다.

# 교훈
ConstraintLayout 를 LinearLayout(vertical) 느낌으로 사용할 때,

ChildView 의 속성 중 아래 두가지는 **다릅니다!!!!**

```xml
android:layout_width="match_parent"
```

```xml
android:layout_width="0dp"
app:layout_constraintStart_toStartOf="parent"
app:layout_constraintEnd_toEndOf="parent"
```





















