# beatmeonthedl



## mitigation

    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments



## Write-up

풀이 : memory leak, unsafe unlink



메뉴 구성을 보면 heap 관련 문제인 것을 알 수 있다.



특이한 점은 커스텀 malloc, free을 사용해서 peda나 pwndbg의 힙 플러그인으로 분석을 못한다.



![image-20200314132432639](D:\private\blog\2017\defcon\pwnable\beatmeonthedl\img\image-20200314132432639.png)



```add_request()``` 에서 malloc(56) 후 128바이트를 read해 overflow가 된다.



![image-20200314132641840](D:\private\blog\2017\defcon\pwnable\beatmeonthedl\img\image-20200314132641840.png)



그리고 free가 자유롭기 때문에 unsafe unlink로 풀 수 있다.



**[주의] 메모리 주소는 디버깅 과정에서 계속 바뀌었습니다..**

![image-20200314134031819](D:\private\blog\2017\defcon\pwnable\beatmeonthedl\img\image-20200314134031819.png)

총 4번 malloc()을 호출했다.



이제 3번째 할당한 청크에 fake chunk를 구성한다.

![fakechunk](D:\private\blog\2017\defcon\pwnable\beatmeonthedl\img\fakechunk.png)

3번째 청크인 0x11310b0부터  FAKE CHUNK를 구성했다.



FAKE CHUNK의 `FD`는  `&target-0x18` = `0x609e78`으로, `BK`에는 `&target-0x10` = `0x609e80`을 덮어썼다. 

그리고 4번째 청크의 `prev_size`를 FAKE CHUNK와의 거리(30)만큼으로 주었고, `SIZE`의 `prev_inuse` bit를 0으로 만들어서 이전청크가 해제된 것처럼 만들었다.



이제 4번째 청크를 해제하면 `prev_inuse` bit가 0이므로 FAKE CHUNK와 병합하게 된다.

병합하는 과정에서 아래와 같은 과정을 거친다.

```
/*
  unlink가 트리거 되면


  chunk3->FAKE_BK->FD = FAKE_FD

  now, ptr1 == (long)&ptr1 - sizeof(long)*3;
  
  // 이때 FAKE_BK->FD 현재 병합하는 청크의 주소와 일치해야함
  // ex) FAKE_BK->FD = 0x609e90, 현재 병합해야하는 청크(FAKE CHUNK)의 P가 *(FAKE_BK->FD)에 있어야 함.

/*
```



여기서 

`chunk3->FAKE_BK->FD` = 0x609e80+0x10

`FAKE_FD` = 0x609e78

![unlink](D:\private\blog\2017\defcon\pwnable\beatmeonthedl\img\unlink.png)

unlink 가 트리거 된 후 `FAKE_FD`가 `FAKE_BK->FD`에 들어갔다.



reqlist는 할당된 포인터를 가리키는 변수다. 바이너리의 `print_list()`함수를 통해 reqlist를 출력할 수 있기 때문에 이 함수를 통해 heap 주소를 leak할 수 있다.



![image-20200314142141538](D:\private\blog\2017\defcon\pwnable\beatmeonthedl\img\image-20200314142141538.png)



leak한 주소(3번째 청크에 덮어씌운 reqlist 주소)에 `update_request()`를 통해 puts@GOT를 overwrite한다.

![puts](D:\private\blog\2017\defcon\pwnable\beatmeonthedl\img\puts.png)



그리고 한번 더 `update_request()`로 첫 번째 청크에 oneshot 가젯을 덮어씌운 후 puts함수를 호출하면 oneshot가젯이 호출되면서 익스가 된다.



![image-20200314143141489](C:\Users\Chanyoung So\AppData\Roaming\Typora\typora-user-images\image-20200314143141489.png)

