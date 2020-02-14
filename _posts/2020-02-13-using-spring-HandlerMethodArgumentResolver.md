---
published: false
---
## HandlerMethodArgumentResolver 활용해 컨트롤러 파라미터 변경 해보기


어느날, 잘 사용하던 Rest Controller의 권한 체크에 이슈가 생겼다.

Rest API에 전달되는 @PathVariable을 이용하여 특정 서비스(DB 조회, 편의상 'A'서비스)를 통해 현재 접근 가능한 API인지 체크해야했다.

여러 API에서 해당 로직이 작동해야했다.

처음에 AOP를 활용하여 비즈니스 로직에 영향을 주지 않는 범위에서 로직을 넣고자 했다.

특정 어노테이션(@Authorised)을 선언하고 이를 포인트 컷으로 지정해 체크하면 되겠다고 생각해서 코드를 넣다가 아쉬운 점을 발겼했다.

어떤 API(편의상 '가' API)의 경우 AOP 내에서 권한 체크에 사용했던  'A'서비스의 DTO를 JSON으로 출력한다.

때문에 '가' API 같은 경우에는 같은 서비스를 2번이나 호출된다.

맘에 좀 안 든다.

차라리 RestController에서 전달 받은 인자로 권한 체크를 하고 이를 비지니스 로직에 넣어주는 것으로 

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTgzMDgxODQyMV19
-->