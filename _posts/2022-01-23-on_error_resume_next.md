---
title: RxJava onErrorResumeNext
date: 2022-01-23 19:00:00 +0900
categories: [Android, RxJava]
tags: [android, rxjava]
img_path: /img/3
---

# onErrorResumeNext 을 사용하면서 겪은 일
아래와 같은 상황이 있었습니다.

ChannelRepositoryImpl.java
```java
public class ChannelRepositoryImpl implements ChannelRepository {
    private final ChannelLocalDataSource local;
    private final ChannelRemoteDataSource remote;

    public Single<Channel> getChannel(String channelId) {
        return fetchChannel(channelId)
                    .onErrorResumeNext(local.getChannel(channelId));
    }

    private Single<Channel> fetchChannel(String channelId) {
        return remote.fetchChannel(channelId)
    }
}
```

간단히 설명하면, getChannel 메소드는

Channel(이하 대화방) 정보를 가져오는 메소드입니다.

remote 에서 정보를 가져오는데 실패한 경우,

local 에 저장된 대화방 정보를 가져오는 일을 하는 메소드입니다.

# 문제 상황
remote 에서 정상적으로 대화방 정보를 가져오는 상황에서는, 

local.getChannel(channelId) 메소드가 실행되길 기대하지 않고 있었습니다.


아래는 local.getChannel(channelId) 메소드의 구현체 부분입니다.

DB 에서 데이터를 가져오는 일을 하죠.

ChannelLocalDataSourceImpl 클래스의 일부입니다.
```java
@Override
public Single<Channel> getChannel(String channelId) {
    return channelDatabase.getChannel(encryptChannelId(channelId))
            .map(Mapper::toChannel)
            .map(this::decryptChannel);
}

private String encryptChannelId(String channelId) {
    Log.d("encrypt channelId : " + channelId);
    return BuildConfig.DEBUG ? channelId : crypto.encrypt(channelId);
}
```
정상적인 상황에서 로그를 찍어보니.. 
```
encrypt channelId : blabla
```

이 로그가 찍히는 것을 확인했습니다.

> 어? fetchChannel 이 정상적으로 동작했는데 Local 구현체 내부의 로그가 왜 찍히지?

onErrorResumeNext 는 스트림 내에서 onError 가 발생했을 때, 

현재 스트림을 바꿔주는 역할을 해준다고 알고 있었습니다.

따라서, 제가 기대하던 상황은 fetchChannel 이 정상적으로 동작할 때는 onErrorResumeNext 는 아무런 일도 하지 않아야 한다고 생각했습니다. 

그래서 로컬 데이터소스 구현체(ChannelLocalDataSourceImpl) 내부의 로그가 찍히는 일은 예상 밖의 일이었죠.

내가 RxJava 를 겉핥기 식으로 알고있구나.. 반성하면서 원인을 찾아봅니다..

## onErrorResumeNext 를 살짝 바꿔보자
기존의 onErrorResumeNext 를 조금만 바꿔보았습니다.
### As-is
```java
return fetchChannel(channelId)
        .onErrorResumeNext(local.getChannel(channelId));
```

### To-be
```java
return fetchChannel(channelId)
        .onErrorResumeNext(throwable -> local.getChannel(channelId));
```

이렇게 수정 하니깐, 로컬 데이터소스 구현체에서 찍히던 아래 로그가 사라졌습니다.

```
encrypt channelId : blabla <- 이 로그가 더이상 안찍힙니다.
```
분명 제가 사용한 onErrorResumeNext 메소드에 차이점이 있어보입니다.

이제 차이점을 한번 알아보겠습니다.

# 원인 찾기 #1
먼저 onErrorResumeNext 메소드 내부를 들여다보려고 합니다.

```java
@CheckReturnValue
@NonNull
@SchedulerSupport(SchedulerSupport.NONE)
public final Single<T> onErrorResumeNext(final Single<? extends T> resumeSingleInCaseOfError) {
    ObjectHelper.requireNonNull(resumeSingleInCaseOfError, "resumeSingleInCaseOfError is null");
    return onErrorResumeNext(Functions.justFunction(resumeSingleInCaseOfError));
}
```
```java
@CheckReturnValue
@NonNull
@SchedulerSupport(SchedulerSupport.NONE)
public final Single<T> onErrorResumeNext(
        final Function<? super Throwable, ? extends SingleSource<? extends T>> resumeFunctionInCaseOfError) {
    ObjectHelper.requireNonNull(resumeFunctionInCaseOfError, "resumeFunctionInCaseOfError is null");
    return RxJavaPlugins.onAssembly(new SingleResumeNext<T>(this, resumeFunctionInCaseOfError));
}
```

이렇게 두가지가 있습니다.

첫번째로는 Single 을 받고, 두번째는 Function 을 받네요.

벌써부터 느낌이 조금 옵니다. 왠지 아직도 절차형 프로그래밍이 머릿 속에 남아있었던게 문제겠네요.

자, 여기서 한번만 더 들어가보겠습니다.

두번째 메소드 내부의 SingleResumeNext 클래스를 한번 구경해보겠습니다.

관련 없는 메소드는 제거했습니다.
```java
public final class SingleResumeNext<T> extends Single<T> {
    final SingleSource<? extends T> source;

    final Function<? super Throwable, ? extends SingleSource<? extends T>> nextFunction;

    public SingleResumeNext(SingleSource<? extends T> source,
            Function<? super Throwable, ? extends SingleSource<? extends T>> nextFunction) {
        this.source = source;
        this.nextFunction = nextFunction;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super T> observer) {
        source.subscribe(new ResumeMainSingleObserver<T>(observer, nextFunction));
    }

    static final class ResumeMainSingleObserver<T> extends AtomicReference<Disposable> implements SingleObserver<T>, Disposable {
        ResumeMainSingleObserver(SingleObserver<? super T> actual,
                Function<? super Throwable, ? extends SingleSource<? extends T>> nextFunction) {
            this.downstream = actual;
            this.nextFunction = nextFunction;
        }

        @Override
        public void onError(Throwable e) {
            SingleSource<? extends T> source;

            try {
                source = ObjectHelper.requireNonNull(nextFunction.apply(e), "The nextFunction returned a null SingleSource.");
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                downstream.onError(new CompositeException(e, ex));
                return;
            }

            source.subscribe(new ResumeSingleObserver<T>(this, downstream));
        }
    }
}
```

조금 길지만 간단하게 살펴보면,

SingleResumeNext 의 subscribeActual 메소드를 통해서 기존 스트림을 구독하는 것으로 보입니다.

여기서 nextFunction 을 통해 에러 상황에서 바꿔줄 스트림을 전달해주고,

본 스트림에서 오류가 발생했을 때 ResumeMainSingleObserver 클래스의 onError 가 트리거되고,

nextFunction 을 apply 하고, 바꿔줄 스트림을 subscribe 하는 모습입니다.

우리의 상황에 대입해보면, fetchChannel 과정에서 exception 이 발생하게 되면,

nextFunction 으로 전달한 Function 을 apply 한 뒤에, 구독하여 

결과적으로 onErrorResumeNext 에 작성한 스트림으로 이어서 진행하게 되겠죠.


# 원인 찾기 #2 (기대대로 동작하는 코드)
기대대로 동작하던 코드를 다시 보겠습니다.

```java
return fetchChannel(channelId)
        .onErrorResumeNext(throwable -> local.getChannel(channelId));
```
여기서 `throwable -> local.getChannel(channelId)` 는 Function 인터페이스를 구현해서 람다로 표현한 모습입니다.

```java
public interface Function<T, R> {
    /**
     * Apply some calculation to the input value and return some other value.
     * @param t the input value
     * @return the output value
     * @throws Exception on error
     */
    R apply(@NonNull T t) throws Exception;
}
```
즉, 우리는 throwable 을 받아서 SingleSource 를 리턴해주는 apply 라는 메소드를 구현한 것입니다.

이 람다를 펼쳐서 작성하면 아래와 같습니다.

```java
@Override
public Single<Channel> getChannel(String channelId) {
    final Function<Throwable, SingleSource<? extends Channel>> resumeFunction = new Function<Throwable, SingleSource<? extends Channel>>() {
            @Override
            public SingleSource<? extends Channel> apply(@NonNull Throwable throwable) throws Exception {
                return local.getChannel(channelId);
            }
        };

    return fetchChannel(channelId)
            .onErrorResumeNext(resumeFunction);
}
```

원인 찾기 #1 에서 봤듯이, onError 가 발생한 뒤에 apply 를 하고 바뀐 스트림을 subscribe 하기 때문에,
fetchChannel 이 정상적으로 동작했을 때는 로컬 데이터소스 구현체에서 아래 로그가 찍히지 않았던 것이죠.

```
encrypt channelId : blabla <-- 이 로그가 찍히지 않습니다.
```

기대대로 동작하는 코드가 왜 그렇게 동작했는지 살펴봤습니다.

# 원인 찾기 #3 (기대대로 동작하지 않았던 코드)
그러면, 처음의 문제상황

```java
return fetchChannel(channelId)
        .onErrorResumeNext(local.getChannel(channelId));
```

이 코드는 fetchChannel 이 정상적인 상황에서도 왜 아래 로그가 찍혔던 것일까요?
```
encrypt channelId : blabla
```

위에서 살펴봤던 Single 을 받는 onErrorResumeNext 메소드에서

Functions.justFunction(resumeSingleInCaseOfError) 를 보겠습니다.
```java
@CheckReturnValue
@NonNull
@SchedulerSupport(SchedulerSupport.NONE)
public final Single<T> onErrorResumeNext(final Single<? extends T> resumeSingleInCaseOfError) {
    ObjectHelper.requireNonNull(resumeSingleInCaseOfError, "resumeSingleInCaseOfError is null");
    return onErrorResumeNext(Functions.justFunction(resumeSingleInCaseOfError));
}
```

여기서 justFunction 을 보면..

```java
public static <T, U> Function<T, U> justFunction(U value) {
    return new JustValue<T, U>(value);
}
```
전달한 value 를 들고 있는 Function 인스턴스 를 만들어 주고 있습니다.

여기서 전달한 value 는 `local.getChannel(channelId)` 입니다.

즉, 에러 상황에서 바꿔줄 스트림(local.getChannel(channelId))을 값으로 갖는

Function 인스턴스를 만들어두기 때문에 로컬 데이터소스 구현체의 로그가 찍히는 것입니다.

물론, 이 바꿔줄 스트림을 apply 하고 subscribe 를 하는 일은 본 스트림에서 onError 가 발생한 뒤의 일입니다.

바꿔줄 스트림을 구독하기 전까지의 일은 fetchChannel 의 오류 여부와 무관하게 진행했고, 그래서 로그가 찍현던 것입니다.

## 조금더 극단적으로 눈에 보이게 표현하면..

```java
@Override
public Single<Channel> getChannel(String channelId) {
    return fetchChannel(channelId)
        .onErrorResumeNext(local.getChannel(channelId));
}
```

위의 코드는 아래와 같습니다.

```java
@Override
public Single<Channel> getChannel(String channelId) {
    return fetchChannel(channelId)
        .onErrorResumeNext(new JustValue<>(local.getChannel(channelId)));
}

static final class JustValue<T, U> implements Callable<U>, Function<T, U> {
    final U value;

    JustValue(U value) {
        this.value = value;
    }

    @Override
    public U call() throws Exception {
        return value;
    }

    @Override
    public U apply(T t) throws Exception {
        return value;
    }
}
```

Function 인터페이스를 구현한 JustValue 인스턴스를 만들어서 onErrorResumeNext 에 넣어주는 일은

본 스트림의 onError 발생 여부와 무관하게 해야하는 일입니다.

JustValue 인스턴스를 만들기 위해서는 local.getChannel(channelId) 메소드는 실행되어야하죠.

물론, 본 스트림에서 에러가 발생 전까지는 구독하지 않구요. 

차가운 옵저버블은 구독하기 전까지는 어짜피 데이터 발행을 하지 않으니까, 

구독하기 전까지의 일은 다해놨다 라고 보면 되지 않을까 싶습니다.

# 정리
onErrorResumeNext(...) 안에 Single 을 넘겨준다면,

해당 Single 을 구독하기 전에 내부적으로 문제가 될 만한 부분이 있는지 체크해봐야겠습니다.


















