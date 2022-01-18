---
title: 자주쓰는 ConstraintLayout 속성(2)
date: 2022-01-17 19:00:00 +0900
categories: [Android, layout]
tags: [android, constraintLayout, layout]
img_path: /img/2
---

# ConstraintLayout
[이전 포스트](https://johnkongdev.github.io/posts/constraint_layout/) 내용을 말로 풀면 아래와 같습니다.
> 이름은 최대한 전부 다 표현해주고, 닉네임의 최소 너비(100dp)는 보장해준다.

이번에는 close 아이콘을 닉네임 옆에 붙어있도록 하는 방법을 알아봅니다.

close 아이콘은 닉네임 옆에 붙어있으면서, 우측 끝에 닿으면 이름과 닉네임이 말줄임 표시가 되고,

close 아이콘은 항상 보여아합니다.

# nickname 의 width 변경

![2_1](2_1.png)

지금은 width 의 남은 영역을 nickname 이 모두 가져가고 있습니다.

close 아이콘을 nickname 바로 옆에 위치시키려면, 

먼저 nickname 의 width 를 wrap_content 로 변경해줍니다.

```xml
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:padding="10dp">

    <TextView
        android:id="@+id/name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        tools:text="홍길동"
        android:maxLines="1"
        android:ellipsize="end"
        app:layout_constrainedWidth="true"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toStartOf="@id/nickname"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/nickname"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        tools:text="으라차차"
        android:maxLines="1"
        android:ellipsize="end"
        app:layout_constraintWidth_min="100dp"
        app:layout_constraintStart_toEndOf="@id/name"
        app:layout_constraintEnd_toStartOf="@id/close"
        app:layout_constraintTop_toTopOf="@id/name"
        android:layout_marginStart="10dp"/>

    <ImageView
        android:id="@+id/close"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@drawable/btn_box_close"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

![2_2](2_2.png)

name 과 nickname 이 모두 wrap_content 속성을 갖고 있어서

남는 너비를 나눠갖는 모습입니다.

코드를 자세히 보면,

name 은 왼쪽으로는 parent, 오른쪽으로는 nickname 과 연결되어있고

nickname 은 왼쪽으로는 name, 오른쪽으로는 close 와 연결되어 있습니다.

그리고 close 는 오른쪽으로 parent 만 연결되어 있습니다.

# close 에 왼쪽 제약 조건 추가

이제는, close 버튼을 단순히 우측에 배치만 하면 안되고 왼쪽 뷰들과 제약관계를 맺어야하기 때문에

close 버튼에 layout_constraintStart_toEndOf 조건을 추가해줍니다.

```xml
<ImageView
    android:id="@+id/close"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:src="@drawable/btn_box_close"
    app:layout_constraintStart_toEndOf="@id/nickname"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintBottom_toBottomOf="parent"/>
```

![2_3](2_3.png)

이제 3개의 뷰가 모두 start, end 로 연결이 되어있습니다.

# 모아서 좌측 정렬

각 뷰들이 좌측으로 모아져서 정렬되도록 하려면,

chain 과 bias 를 알아야합니다.

[Android Developer chain-style 가이드](https://developer.android.com/reference/androidx/constraintlayout/widget/ConstraintLayout#chain-style)

우리가 필요한 기능은 Packed Chain with Bias 로 표현이 가능할 것 같습니다.

```xml
<TextView
    android:id="@+id/name"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    tools:text="홍길동"
    android:maxLines="1"
    android:ellipsize="end"
    app:layout_constrainedWidth="true"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toStartOf="@id/nickname"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintHorizontal_chainStyle="packed"
    app:layout_constraintHorizontal_bias="0"/>
```

제약 조건의 첫번째 뷰인 name 에 horizontal chainStyle 을 packed 로 주고,

bias 를 0으로 설정합니다.

![2_4](2_4.png)

packed 속성으로 뷰들이 모아지고, bias 속성으로 좌우 균형을 깨고, 왼쪽으로 몰아두었습니다.

bias 속성은 0~1 사이 값을 넣어주며, 0 이면 좌측, 1 이면 우측으로 붙습니다.

그 사이값을 실무에서 사용할 일은 없었습니다.

# 닉네임의 layout_constraintWidth_min 제거
닉네임 옆에 바로 붙여야하는데, layout_constraintWidth_min 설정 때문에 

닉네임과 close 사이에 여백이 보여서 제거해줍니다.

```xml
<TextView
    android:id="@+id/nickname"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    tools:text="으라차차으라차차"
    android:maxLines="1"
    android:ellipsize="end"
    app:layout_constraintStart_toEndOf="@id/name"
    app:layout_constraintEnd_toStartOf="@id/close"
    app:layout_constraintTop_toTopOf="@id/name"
    android:layout_marginStart="10dp" />
```

![2_5](2_5.png)

![2_6](2_6.png)

![2_7](2_7.png)

몇가지 케이스에서는 이상적으로 동작하는데, 

닉네임을 wrap_content 로 변경하면서 닉네임이 길어지는 경우 이름 영역까지 잡아먹는 현상이 발생합니다.

선택에 따라 간단히 대응할 수 있습니다.

## 1. name 에 min width 를 주는 방법
name 속성에 최소너비를 주변 간단히 해소 가능합니다.

다만, 이름이 외자거나.. 예외상황에서 자칫 이름과 닉네임 간격이 다르게 보이면 사용자가 이질감을 느낄 수 있습니다.

## 2. nickname 에 max width 를 주는 방법
layout_constraintWidth_max 또는 maxEms 등을 사용하는 것도 한 방법입니다.

## 3. Depth 를 하나 추가하는 방법
이전 포스트에서 nickname 에 min width 를 100dp 로 설정했던 것을 활용하는 방법입니다.

nickname 과 close 버튼을 붙여야해서, 위에서 nickname 의 min width 를 제거했는데,

nickname 과 close 버튼을 묶어서 또 다른 ConstraintLayout 에 넣어주고,

ConstraintLayout 의 min width 를 120dp(100dp + close 아이콘 크기) 로 설정하는 방법입니다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:padding="10dp">

    <TextView
        android:id="@+id/name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        tools:text="홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동"
        android:maxLines="1"
        android:ellipsize="end"
        app:layout_constrainedWidth="true"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toStartOf="@id/nickname_close_container"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintHorizontal_chainStyle="packed"
        app:layout_constraintHorizontal_bias="0" />

    <androidx.constraintlayout.widget.ConstraintLayout
        android:id="@+id/nickname_close_container"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:layout_constraintWidth_min="120dp"
        app:layout_constraintStart_toEndOf="@id/name"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="@id/name"
        android:layout_marginStart="10dp" >

        <TextView
            android:id="@+id/nickname"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            tools:text="으라차차"
            android:maxLines="1"
            android:ellipsize="end"
            app:layout_constrainedWidth="true"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toStartOf="@id/close"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintHorizontal_chainStyle="packed"
            app:layout_constraintHorizontal_bias="0"/>

        <ImageView
            android:id="@+id/close"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:src="@drawable/btn_box_close"
            app:layout_constraintStart_toEndOf="@id/nickname"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintBottom_toBottomOf="parent" />
        
    </androidx.constraintlayout.widget.ConstraintLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```

코드를 보면,

내부에 추가한 ConstraintLayout 안에 chainStyle 과 bias 로 

닉네임과 close 버튼을 묶어서 좌측으로 보냈습니다.

그리고 이 내부의 ConstraintLayout 에 layout_constraintWidth_min 를 지정했습니다.

몇가지 상황에서 아래와 같이 동작합니다.

![2_8](2_8.png)

![2_9](2_9.png)

![2_10](2_10.png)

![2_11](2_11.png)

![2_12](2_12.png)

![2_13](2_13.png)

# 정리
아직 ConstraintLayout 속성을 100% 이해하지 못한 점도 있고,

저런 상황에서 완벽히 동작하려면 동적으로 핸들링을 해야만 하나? 싶은 생각도 듭니다.

더 알아봐야합니다..만 이 정도로도 코너케이스를 제외하면 충분히 커버가 가능해보입니다.

혹시 더 나은 활용법이 있으면 알려주세요!













