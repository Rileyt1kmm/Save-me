---
title: 파이널라이저
content_type: concept
weight: 80
---

<!-- overview -->

{{<glossary_definition term_id="finalizer" length="long">}}

파이널라이저(Finalizer)를 사용하면 리소스를 삭제하기 전 특정 정리 작업을 수행하도록
{{<glossary_tooltip term_id="controller">}}에 경고하여
리소스의 {{<glossary_tooltip term_id="garbage-collection">}}을 제어할 수 있다.

파이널라이저는 보통 실행할 코드를 지정하지 않는다.
대신 파이널라이저는 일반적으로 어노테이션과 비슷하게 특정 리소스에 대한 키들의 목록이다.
일부 파이널라이저는 쿠버네티스가 자동으로 지정하지만,
사용자가 직접 지정할 수도 있다.

## 파이널라이저의 작동 방식

매니페스트 파일을 사용해 리소스를 생성하면
`metadata.finalizers` 필드에 파이널라이저를 명시할 수 있다.
리소스를 삭제하려 할 때는
삭제 요청을 처리하는 API 서버가 `finalizers` 필드의 값을 인식하고 다음을 수행한다.

  * 삭제를 시작한 시각과 함께 `metadata.deletionTimestamp` 필드를 추가하도록
  오브젝트를 수정한다.
  * 오브젝트의 `metadata.finalizers` 필드가 비워질 때까지 오브젝트가 제거되지 않도록 한다.
  * `202` 상태 코드를 리턴한다(HTTP "Accepted").

이 파이널라이저를 관리하는 컨트롤러는 `metadata.deletionTimestamp`를 설정하는 오브젝트가 업데이트 되었음을 인지하여
오브젝트의 삭제가 요청되었음을 나타낸다.
그런 다음 컨트롤러는 그 리소스에 지정된 파이널라이저의 요구사항을 충족하려 시도한다.
컨트롤러는 파이널라이저 조건이 충족될 때 마다
리소스의 `finalizers` 필드에서 해당 키(key)를 제거한다.
`finalizers` 필드가 비워지면 `deletionTimestamp` 필드가 설정된 오브젝트는 자동으로 삭제된다.
또한 파이널라이저를 사용하여 관리되지 않는 리소스가 삭제되지 않도록 할 수 있다.

파이널라이저의 일반적인 예로는 `퍼시스턴트 볼륨(Persistent Volume)` 오브젝트가 실수로 삭제되는 것을 방지하는 `kubernetes.io/pv-protection`가 있다.
파드가 `퍼시스턴트 볼륨` 오브젝트를 사용 중일 때
쿠버네티스는 `pv-protection` 파이널라이저를 추가한다.
`퍼시스턴트 볼륨`을 삭제하려 하면 `Terminating` 상태가 되지만
파이널라이저가 존재하기 때문에 컨트롤러가 삭제할 수 없다.
파드가 `퍼시스턴트 볼륨`의 사용을 중지하면
쿠버네티스가 `pv-protection` 파이널라이저를 해제하고 컨트롤러는 볼륨을 삭제한다.

## 소유자 참조, 레이블, 파이널라이저 {#소유자-레이블-파이널라이저}

{{<glossary_tooltip term_id="label">}}와 마찬가지로
쿠버네티스에서 
[소유자 참조(Owner reference)](/docs/concepts/overview/working-with-objects/owners-dependents/)는
오브젝트 간의 관계를 설명하지만 다른 목적으로 사용된다.
{{<glossary_tooltip term_id="controller">}}가 파드와 같은 오브젝트를 관리할 때
레이블을 사용하여 관련 오브젝트의 그룹에 대한 변경 사항을 추적한다.
예를 들어 {{<glossary_tooltip term_id="job">}}이 하나 이상의 파드를 생성하면
잡 컨트롤러는 해당 파드에 레이블을 적용하고
클러스터 내 동일한 레이블을 갖는 파드에 대한 변경 사항을 추적한다.

또한, 잡 컨트롤러는 이러한 파드에 *소유자 참조*도 추가하여 파드를 생성한 잡을 가리킨다.
이 파드가 실행될 때 잡을 삭제하면
쿠버네티스는 사용자 참조(레이블 대신)를 사용하여
클러스터 내 어떤 파드가 정리되어야 하는지 결정한다.

쿠버네티스는 또한 삭제 대상 리소스에 대한 소유자 참조를 식별할 때 
파이널라이저를 처리한다.

경우에 따라 파이널라이저는 종속 오브젝트의 삭제를 차단할 수 있으며
이로 인해 대상 소유자 오브젝트가 
완전히 삭제되지 않고 예상보다 오래 유지될 수 있다.
이 경우 대상 소유자 및 종속 객체에 대한 
파이널라이저와 소유자 참조를 확인해 원인을 해결해야 한다.

{{<note>}}
오브젝트가 삭제 상태에 있는 경우, 삭제를 계속하려면 파이널라이저를 수동으로 제거해서는 안 된다.
일반적으로 파이널라이저는 특정한 목적으로 가지고 리소스에 추가되므로,
강제로 제거하면 클러스터에 문제가 발생할 수 있다.
이는 파이널라이저의 목적을 이해하고
다른 방법(예를 들어, 일부 종속 객체를 수동으로 정리하는 것)으로
수행될 때만 수행해야 한다.
{{</note>}}

## {{% heading "whatsnext" %}}

* 쿠버네티스 블로그에서 
[파이널라이저를 사용해 삭제 제어하기](/blog/2021/05/14/using-finalizers-to-control-deletion/)를 읽어본다.
