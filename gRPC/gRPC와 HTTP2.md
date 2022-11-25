# gRPC와 HTTP2

- gRPC가 어떻게 HTTP/2의 Connection을 활용해 서로 다른 서비스들이 효과적이며 scale 가능하게끔 해주는지 알아보자.  
  HTTP/1.1과 HTTP/2의 차이점은 [여기](https://github.com/sang-w0o/Study/blob/master/Others/HTTP1.1_HTTP2.0.md)에서 확인할 수 있다.

## gRPC의 기본 개념

- 본격적으로 들어가기 전에, 우선 HTTP/2의 개념과 연관된 gRPC의 개념을 보자.  
  gRPC는 **Channels**, **RPC(Remote Procedure Call)**, 그리고 **Messages** 라는 3개의 개념을 소개한다.  
  이 세가지 사이의 관계는 간단한데, 각 channel은 여러 RPC를 가질 수 있으며 각 RPC는 여러 message를 가질 수 있다.

![picture 1](/images/GRPC_HTTP2_1.png)

- 이제 gRPC가 어떻게 HTTP/2와 연관되는지를 보자.

![picture 2](/images/GRPC_HTTP2_2.png)

> - gRPC channel은 HTTP/2 Connection을 사용하며, RPC는 HTTP/2 Connection 상에 스케쥴링된다.  
>   그리고 RPC 내의 message들은 HTTP/2 Frame으로 전달된다.
> - HTTP/2는 Connection 내의 여러 개의 Stream이, Stream 내에 여러 개의 Message들이, 그리고 각 Message 내에는  
>   여러 개의 Frame들이 존재한다.

- gRPC Channel은 gRPC의 핵심 개념이다. HTTP/2의 Stream은 하나의 connection 상에서 동시에 여러 요청과 응답을 할 수 있도록 해주는데,  
  gRPC의 Channel은 동시에 맺어진 여러 connection에 여러 stream을 만들 수 있게 해주면서 stream의 개념을 확장한다.  
  겉으로 봤을 때 channel은 사용자에게 message를 전달할 수 있는 단순한 인터페이스를 제공하지만, 내부적으로는 해당 channel이 사용할  
  connection들을 살려놓고, healthy하게, 그리고 사용가능하도록 하기 위해 엄청나게 복잡한 작업들이 수행된다.

- gRPC Channel은 하나의 endpoint에 대한 virtual connection이라 할 수 있고, 실제로는 수많은 HTTP/2 Connection들을 가진다.  
  반면 gRPC RPC는 특정 connection과 연관 관계를 가지는데, RPC는 실제로 HTTP/2 Stream으로 구현된다.  
  또한 gRPC Message는 gRPC RPC와 연관 관계를 가지고, HTTP/2 Data Frame으로 구현된다. 조금 더 자세히 말하자면 message는  
  data frame의 상위에 존재한다. Data frame은 여러 개의 gRPC Message들을 포함할 수 있으며 만약 gRPC Message의 크기가  
  상당히 크다면 여러 개의 Data frame을 사용할 수 있다.

## Resolvers and Load Balancers

- gRPC는 connection을 alive, healthy, utilized 상태로 두기 위해 대표적으로 **Resolver**와 **Load Balancer**를 제공한다.  
  Resolver는 DNS를 실제 IP 주소(들)로 바꿔주며, 이를 Load Balancer로 전달한다. 그 후 Load Balancer는 Resolver에게 전달받은  
  주소들에 대해 RPC들을 Load Balancing하고 connection을 수립하게 해준다.

![picture 3](/images/GRPC_HTTP2_3.png)

- 예시 상황을 보자. 예를 들어 DNS Resolver가 host name의 IP 주소를 13개 찾았다 하면, RoundRobin Balancer는 13개의 주소에 대해  
  각각 1개씩, 총 13개의 connection을 수립하고 RPC들을 이 connection들에 대해 round robin 할 것이다.  
  조금 더 간단한 load balancer의 경우에는 13개 중 처음으로 찾은 IP 주소에 대해서만 connection을 맺을 수도 있다.  
  또다른 상황으로 DNS Resolver가 파악할 수 있는 IP 주소가 1개 밖에 없다는 것을 알고, 사용자가 여러 개의 connection이  
  필요한 상황이라면 load balancer가 해당 IP 주소에 대해 다수의 connection을 미리 맺어두도록 할 수도 있다.

---

## Connection Management

- 설정이 완료된 gRPC는 connection들의 pool을 Resolver와 LB를 사용해 healthy, alive, utilized한 상태로 유지하게 된다.

- Connection이 실패하면 Load Balancer는 마지막으로 connection 수립에 성공한 주소들에 대해 reconnect를 하기 시작할 것이다.  
  그와 동시에 resolver는 host name의 IP들을 다시 알아내려 할 것이다. 이러한 로직은 많은 시나리오에서 유용하다.  
  예를 들어 만약 proxy가 더 이상 유효하지 않다면, Resolver가 IP 주소들의 목록을 갱신해 proxy의 주소는 포함하지 않게끔 할 수 있다.  
  또다른 예시로 DNS entry는 시간이 지나면 바뀔 수 있기에 IP 주소들의 목록을 주기적으로 갱신시키도록 할 수도 있다.  
  이런 과정은 gRPC가 장기적으로 회복 가능한 성격을 가지는 것을 보여준다.

- Resolver가 IP 주소들을 알아낼 때마다 새로운 IP 주소들은 Load Balancer에게 전달된다. 새로운 IP 주소들을 받은 Load Balancer는  
  기존의 주소와 비교해 새로운 주소 목록에는 없는 주소들에 대해서 connection을 끊고, 기존에는 없던 새로운 주소들에 대해서는 connection을 맺는다.

---

## Identifying Failed Connections

- gRPC의 connection 관리 방식 효율성은 gRPC가 connection failure를 파악하는 방식에 달려있다.  
  일반적으로는 아래 2가지의 connection 실패 상황이 있다.

  - _Clean failure_ : 실패가 서로에게 알려지는 상황
  - _Less-clean failure_ : 실패가 서로에게 알려지지 않는 상황

- 첫 번째로 알기 쉬운 _Clean failure_ 를 보자. Clean failure은 endpoint에서 의도적으로 connection을 죽일 때 발생할 수 있다.  
  예를 들어 타이머가 끝났거나 endpoint의 서비스가 종료되는 상황 등의 이유로 endpoint 측에서 connection을 닫을 수 있다.  
  이렇게 _clean_ 하게 닫히는 connection들은 TCP 기본 절차를 따르게 되는데, TCP의 `FIN Handshake` 절차를 수행한다.
  이렇게되면 우선 HTTP/2 Connection이 끊기며, 이어서 gRPC Connection도 끊긴다. 이후 gRPC는 즉각적으로 reconnect를 시도할 것이다.  
  _Clean_ 한 만큼, 이러한 경우에는 추가적으로 HTTP/2 또는 gRPC 작업이 필요 없다.

- _Less Clean Failure_ 는 모종의 이유로 endpoint가 죽어 client에게 실패를 알리지 못하고 connection이 죽게되는 경우이다.  
  이 경우에 TCP는 최대 10분 동안 지속적으로 retry를 하고, 결국 연결에 실패하면 connection이 실패했다고 판단하게 된다.  
  최대 10분이라는 긴 시간 후에 connection이 실패했다고 판단하는 것은 매우 비효율적이다. 이를 해결하기 위해 gRPC는 HTTP/2의 개념을  
  활용하는데, KeepAlive를 설정해 놓은 gRPC라면 gRPC는 주기적으로 `HTTP/2 PING frame`들을 보낸다. 이 frame은 flow control을  
  건너뛰며 오로지 connection이 alive 한지만을 판별하기 위한 것이다. 만약 PING에 대한 응답이 적절한 시간 내에 오지 않는다면,  
  gRPC는 해당 connection이 fail했다고 판단하고, 위에서 설명한대로 reconnect를 시도하기 시작한다.

- 이러한 방식들로 gRPC는 connection 들의 pool을 healthy하게 유지하고, HTTP/2를 사용해 connection들의 health를 주기적으로 파악한다.  
  위의 모든 과정들은 user(gRPC를 호출하는 클라이언트)에게는 숨겨진다. message가 다른 IP로 redirect되는 과정은 자동으로, 매우 짧은 시간 내에  
  진행되기 때문에 user 입장에서는 항상 healthy한 connection을 사용하는 것처럼 보인다.

---

## Keeping Connections Alive

- 위에서 본 것처럼 KeepAlive는 주기적으로 `HTTP/2 PING frame`을 보내 connection이 활성화(alive) 되어 있는지를 판단하기에  
  fail된 connection들을 빠르게 알아낼 수 있다는 장점이 있다. 여기에 더해 또다른 장점이 있는데, 바로 proxy들에게 자신이  
  사용 가능한 상태임을 알리는 것이다.

- 클라이언트가 proxy를 통해 서버에 데이터를 전송하는 상황을 예시로 생각해보자. 만약 proxy가 없다면 서버와 클라이언트는 connection을  
  무기한으로 수립하고, 필요할 때마다 데이터를 보낼 수도 있다. 하지만 proxy의 경우, 보통 proxy는 리소스에 제약이 있기에 유휴(idle)한  
  connection들은 리소스를 낭비하지 않기 위해 close하게 된다. GCP의 load balancer는 기본적으로 10분 동안 idle한 connection은  
  끊고, AWS의 경우에는 60초 동안 idle하다면 끊는다.

- gRPC는 connection의 상태 파악을 위해 `HTTP/2 PING frame`을 주기적으로 보내기에 모든 connection은 non-idle하게 된다.  
  따라서 위에서 idle connection을 kill하는 방식을 사용하는 endpoint들은 gRPC connection들은 kill하지 못하게 된다.

---

- 이 문서는 [이 문서](https://grpc.io/blog/grpc-on-http2/)의 번역본 입니다.
