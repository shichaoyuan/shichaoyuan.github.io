---
layout: post
title: Slick中的upsert操作
date: 2015-08-13 08:11:30
tags:
 - scala
 - slick
tags:
---

##1

在做一个Single Page Application，然后使用三方登陆，不维护账号信息，每次登陆拉一次最新的三方账户信息。

所以数据库操作就有个问题，没有注册，insert，老用户，update，俗称upsert。

##2

使用mysql直接写SQL语句很简单：

[`INSERT ... ON DUPLICATE KEY UPDATE `](http://dev.mysql.com/doc/refman/5.6/en/insert-on-duplicate.html)

##3

但是这个小项目使用[`play-slick 1.0.1`](https://www.playframework.com/documentation/2.4.x/PlaySlick)

刚开始学着用，就捣鼓出了这样一段逻辑

```scala

def insert(user: User): Future[Int] = db.run(Users += user)
  
def update(user: User): Future[Int] = {
  db.run(Users.filter(_.id === user.id.get).update(user))
}
  
def selectByQQOpenID(qqOpenID: String): Future[Option[User]] = db.run( Users.filter(_.qqOpenID === qqOpenID).result.headOption )



def createOrUpdateUserByOpenID(openID: String, userInfo: UserInfo): Future[Int] = {
	val user = userDAO.selectByQQOpenID(openID)
	user.flatMap { 
		case Some(u) => userDAO.update(u.copy(name = userInfo.name, headURL = userInfo.headURL))
		case None => userDAO.insert(User(None, userInfo.name, userInfo.headURL, openID))
	}
}
```

##4

感觉不漂亮，搜了一下，原来还挺多研究这个问题的……

比如：[Upsert in Slick 3](http://underscore.io/blog/posts/2015/07/14/upsert.html)

原来有个内建的`insertOrUpdate `，代码提示居然没让我看到……

所以对于单主键

```scala
def postReview(title: String, rating: Int): DBIO[Int] = for {
  existing <- reviews.filter(_.title === title).result.headOption
  row       = existing.map(_.copy(rating=rating)) getOrElse Review(title, rating)
  result  <- reviews.insertOrUpdate(row)
} yield result
```

对于联合主键

```scala
def postReview(critic: String, title: String, rating: Int): DBIO[Int] = for {
  existing <- reviews.filter(r => r.title === title && r.critic === critic).result.headOption
  row       = existing.map(_.copy(rating=rating)) getOrElse Review(critic, title, rating)
  result  <- reviews.insertOrUpdate(row)
} yield result
```

当然也可以纯手工

```scala
def postReview(critic: String, title: String, rating: Int)(implicit ec: ExecutionContext): DBIO[Int] = {
  for {
    rowsAffected <- reviews.filter(r => r.critic === critic && r.title === title).map(_.rating).update(rating)
    result <- rowsAffected match {
      case 0 => reviews += Review(critic, title, rating)
      case 1 => DBIO.successful(1)
      case n => DBIO.failed(new RuntimeException(s"Expected 0 or 1 change, not $n for $critic @ $title"))
    }
  } yield result
}
```



