---
layout: post
title:  "JPA에서 순환참조를 해결하는 방법 | @JsonIgnore, @JsonManagedReference, @JsonBackReference"
date:   2016.12.11 15:36:00 +0900
categories: jpa java
---
JPA는 ORM이기 때문에 RDB를 관리하는데 있어서 양방향 참조를 필요로 한다. 

물론 필수는 아니지만 Entity는 본질적인 데이터를 표현하는 것이기 때문에 그 관계에 대하여 명세해주는 것이 원칙이라고 생각한다. (이에 대해 별개로 처리하는 부분이 있다면 DTO를 통하는 방법이 맞다고 생각한다.) 

이러한 양방향 참조로 인해서 RESTFul API 서버를 구현하는데 있어서 문제가 생겼는데 바로 응답에 Entity를 담아서 보낼 경우에 JsonSerializer가 toString()을 호출할 때 property들을 매핑하는 과정에서 무한 순환 참조가 일어나게 되는 문제이다. 

예를 들어 다음과 같다.

```java
public class Article { 
    @Id private long id;
    // ...
    @OneToMany(mappedBy = "article", fetch = FetchType.LAZY)
    private List comments;
} 

public class Comment {
    @Id private long id;
    // ...
    @ManyToOne(fetch = FetchType.LAZY) 
    @JoinColum(name = "article_id") 
    @LazyToOne(value = LazyToOneOption.NO_PROXY) 
    private Article article;
}
```

위와 같은 형태로 글(Article)은 다수의 댓글(Comment)객체를 갖고 있고 댓글의 경우 그 반대이다. RDB의 특성상 이 관계의 참조의 키가 되는 article_id는 comment 테이블이 갖고 있는 형태이나 ORM의 특성상 이러한 관계를 양쪽 클래스 모두에 정의하게 된다.

이 상태에서 Article 클래스의 toString()을 호출하게 되면 LazyFetch라고 하더라도 모든 클래스의 property를 순회하게 되며, 이 때 comments를 조회하게 되고 Hibernate(혹은 다른 JPA 구현체)는 데이터베이스에 질의하고 데이터를 가져온다. 이 때 Article.getcomments() -> Comment.getArticle() -> Article.getComments() -> ... 와 같이 객체가 서로를 무한으로 호출하게 되는 문제가 있다. 

이러한 문제를 해결하는 방법으로 @JsonIgnore를 사용하는 방법이 있고 @JsonManagedReference와 @JsonBackReference를 사용하는 방법이 있다. 표면적으로 두 방법 모두 순환참조를 방어하는 형태이지만 본질적으로 약간의 차이가 있다. @JsonIgnore의 경우는 실제로 property에 null을 할당하는 방식이고 @JsonManagedReference와 @JsonBackReference는 본질적으로 순환참조를 방어하기 위한 Annotation이다. 

정리를 하자면 json serialize 과정에서 null로 세팅하고자 하면 @JsonIgnore 사용하면 되고, 순환참조에 대한 문제를 해결하고자 한다면 부모 클래스측에 @JsonManagedReference를, 자식측에 @JsonBackReference를 Annotation에 추가해주면 된다.

기본적으로 DTO와 같이 별도의 전달 객체를 활용하여 Mapper를 이용한다면 위와 같은 문제가 없겠지만 경우에 따라 필요한 옵션이 있을 수 있어 찾아보았다.


참조 : [stackoverflow.com/a/37393711][ref1]

[ref1]: http://stackoverflow.com/a/37393711
