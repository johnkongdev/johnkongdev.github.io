---
title: 자주쓰는 ConstraintLayout 속성
date: 2022-01-17 09:00:00 +0900
categories: [Android, xml]
tags: [android, constraintlayout, layout]
img_path: /img/1
---

# ConstraintLayout
레거시한 코드가 아니라면 레이아웃 파일에 꼭 들어가는 ConstraintLayout.

일반적인 화면 구성에는 어려울 것 없지만,
간혹 복잡한 레이아웃(예를 들면, 가변 텍스트가 수평배치되는 등..)에서는
조금 더 잘 알고 쓰면, 

Layout depth 를 최대한 적게 유지할 수 있습니다.

# 이름, 별명, 아이콘이 수평 배치
![1_1](1_1.png)

이런 화면을 구성한다고 생각해봅시다.

```xml
<TextView
    android:id="@+id/name"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    tools:text="홍길동"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent"/>

<TextView
    android:id="@+id/nickname"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    tools:text="으라차차"
    app:layout_constraintStart_toEndOf="@id/name"
    app:layout_constraintTop_toTopOf="@id/name"
    android:layout_marginStart="10dp"/>

<ImageView
    android:id="@+id/close"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:src="@drawable/btn_box_close"
    app:layout_constraintRight_toRightOf="parent"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintBottom_toBottomOf="parent"/>
```
상위의 ConstraintLayout 을 생략하면 대략 이런 형태가 될겁니다.

그런데, 문제가 있습니다.

![1_2](1_2.png)

이렇게 이름이 긴 경우, 또는 닉네임이 긴 경우 대비를 해야겠습니다.

# name 영역의 좌우 제약 정리
기존에는 start 제약만 추가되어있습니다.

nickname 과의 연결을 위해
constraintEnd_toStartOf 조건을 추가해줍니다.

한줄로 표현해주려고, maxLines 옵션도 추가합니다.

```xml
<TextView
    android:id="@+id/name"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    tools:text="홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동"
    android:maxLines="1"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toStartOf="@id/nickname"
    app:layout_constraintTop_toTopOf="parent" />
```

# nickname 영역의 좌우 제약 정리
마찬가지로, nickname 영역도 end 제약을 추가해줍니다.
```xml
<TextView
    android:id="@+id/nickname"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    tools:text="으라차차"
    app:layout_constraintStart_toEndOf="@id/name"
    app:layout_constraintEnd_toStartOf="@id/close"
    app:layout_constraintTop_toTopOf="@id/name"
    android:layout_marginStart="10dp"/>
```

여기까지 진행하면 아래와 같이 표현됩니다.

![1_3](1_3.png)

## name 영역 정리
한줄로 잘 배치가 됐지만, 이름의 끝이 잘렸으나 인지하기 어렵습니다.

ellipsize 옵션을 넣어서 말줄임 표시를 추가해줍니다.

```xml
<TextView
    android:id="@+id/name"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    tools:text="홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동홍길동"
    android:maxLines="1"
    android:ellipsize="end"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toStartOf="@id/nickname"
    app:layout_constraintTop_toTopOf="parent" />
```
![1_4](1_4.png)

# nickname 영역이 길어지면?
닉네임 영역이 길어지면, 이름이 사라지게 됩니다.

이유는 nickname 뷰가 wrap_content 를 사용하고 있기 때문입니다.

![1_5](1_5.png)

nickname 의 width 를 0dp 로 바꿔주고,
줄바꿈을 방지하기위해 maxLines 와 ellipize 옵션도 추가해줍니다.

```xml
<TextView
    android:id="@+id/nickname"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    tools:text="으라차차으라차차으라차차으라차차으라차차으라차차으라차차"
    android:maxLines="1"
    android:ellipsize="end"
    app:layout_constraintStart_toEndOf="@id/name"
    app:layout_constraintEnd_toStartOf="@id/close"
    app:layout_constraintTop_toTopOf="@id/name"
    android:layout_marginStart="10dp"/>
```

![1_6](1_6.png)

자, 이제 가변인 name 과 nickname 이 길어져도 한 줄에 잘 표현이 됩니다.

사실 흔한 케이스는 아니지만, 이름이 긴 외국인이 사용하거나, nickname 에 다른 정보들을 넣어서 내용이 길어지는 경우를 잘 핸들링 할 수 있습니다.

# name 이 짧아질 때?!

![1_7](1_7.png)

이름이 짧아졌을 때 여백이 생겼습니다.

name, nickname 이 모두 0dp width 와 동일한 제약조건으로 연결되어있으니

어찌보면, 공평하게 나눠갖는게 맞습니다.

이름 옆에 닉네임이 붙어서 보이는게 일반적이니, 

이름을 다시 wrap_content 로 수정합니다.

단, wrap_content 속성만 사용한다면 이름이 길어질 때 길어진대로 이름을 감싸기만 합니다.

즉, 화면을 벗어난채로 쭉쭉 길어만 지게 되죠.

그래서 layout_constrainedWidth : true 조건을 넣어줍니다.

보통은 wrap 으로 보여주고, 제약 조건이 필요한 시점에는 마치 0dp 로 설정했을 때 처럼 동작하게해서 화면 밖으로 벗어나는 것을 막아줍니다.

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
    app:layout_constraintTop_toTopOf="parent" />
```

![1_8](1_8.png)

이름 옆으로 닉네임이 잘 붙어서 보입니다.

다만, 조건을 변경한 채로 다시 이름을 길게 설정하면

닉네임의 공간이 확보가 안되는 문제가 생깁니다.

몇가지 방법이 있겠지만, 보통 정보의 우선순위가 이름 > 닉네임 이라고 생각해서

닉네임의 최소 width 를 보장하는 정도로 처리합니다.

layout_constraintWidth_min 을 100dp 정도로 설정해줍니다.
```xml
<TextView
    android:id="@+id/nickname"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    tools:text="으라차차"
    android:maxLines="1"
    android:ellipsize="end"
    app:layout_constraintWidth_min="100dp"
    app:layout_constraintStart_toEndOf="@id/name"
    app:layout_constraintEnd_toStartOf="@id/close"
    app:layout_constraintTop_toTopOf="@id/name"
    android:layout_marginStart="10dp"/>
```

# 정리
![1_9](1_9.png)

![1_10](1_10.png)

![1_11](1_11.png)

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
        tools:text="홍길동"
        android:maxLines="1"
        android:ellipsize="end"
        app:layout_constrainedWidth="true"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toStartOf="@id/nickname"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/nickname"
        android:layout_width="0dp"
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
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

가로로 배치되는 텍스트의 가변 케이스를 ConstraintLayout 으로 표현해봤습니다.

다음에는, close 아이콘이 닉네임의 우측에 붙어 있는 방법을 알아보겠습니다!
