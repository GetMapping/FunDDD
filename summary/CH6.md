1. 스프링 시큐리티에서 권한 체크
   @PreAuthorize와 config 중 선호도.
   - config : 복잡해지면 몰아넣는게 좋을듯.
   - @PreAuthorize : 이미지에 따라 다르게 수행해야할 때. 메서드 단위에서 수행해야할 때.

2. 파사드 : 클라이언트 위한 인터페이스 만들어주자.
 컨트롤러마다 서비스를 1:1로 만들어줘야하나?
 컨트롤러 1 : 서비스 다 등..
ex. 조직도 개발 - 서비스 한 개를 컨트롤러 여러군데에서 쓰면 코드 2번 짤 일 없고 좋을듯.

https://velog.io/@bagt/Design-Pattern-Facade-Pattern-%ED%8D%BC%EC%82%AC%EB%93%9C-%ED%8C%A8%ED%84%B4
https://velog.io/@jojogreen91/Facade-Pattern
https://logical-code.tistory.com/211
