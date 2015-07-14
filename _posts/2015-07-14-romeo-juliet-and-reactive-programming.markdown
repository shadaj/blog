---
layout: post
title: "Romeo, Juliet, and Reactive Programming"
date: "2015-07-14"
color: crimson
---

I recently finished reading the play *Romeo and Juliet* by William Shakespeare in my literature class. With the final exam coming up I was rereading the play over and over again, deepening my understanding of every part of the plot. Cramming the play into my head led to a crazy idea -- what if *Romeo and Juliet* was modeled with actors? The next day this still seemed like a good idea and so began my journey of implementing *Romeo and Juliet* with Scala and Akka.

## Background
If you don't know how *Romeo and Juliet* proceeds, be cautioned as this post has many spoilers. If you haven't read the play, and want to take a shortcut, you can read it at [SparkNotes NFS](http://nfs.sparknotes.com/romeojuliet/), the digital version of *No Fear Shakespeare*, which provides the original text and an English translation (I'm kidding, it's just simplified language).

## The Romance
When it comes to Romeo and Juliet, the two most important actors are **Romeo** and **Juliet**, so let's implement them as such.

```scala
class Romeo extends Actor {
  def receive: Receive = {}
}

class Juliet extends Actor {
  def receive: Receive = {}
}
```

We need to make Romeo and Juliet fall in love with each other, so we create a message `FallInLove` that takes an `ActorRef` to fall in love with. This message triggers a context change in the recipient to enter the `loving` state.

```scala
case class FallInLove(lover: ActorRef)

class Romeo extends Actor {
  def receive: Receive = {
    case FallInLove(lover) =>
      context.become(loving(lover))
  }

  def loving(lover: ActorRef): Receive = {}
}

class Juliet extends Actor {
  def receive: Receive = {
    case FallInLove(lover) =>
      context.become(loving(lover))
  }

  def loving(lover: ActorRef): Receive = {}
}
```

Now that Romeo and Juliet are in love with each other, they need to express their feelings. We will have the actors ask each other with the `DoYouLoveMe` message, expecting (hopefully) an `ILoveYou` message in response. We use the ask pattern from Akka to handle responses to messages as futures. In order to use this pattern we are required to set a timeout that represents how long to wait for a response, which as our lovers have little patience, we set to five seconds. Whenever either of the actors receive a `DoYouLoveMe` message, they immediately respond with `ILoveYou` and schedule asking the other actor `DoYouLoveMe` after one second. What an insecure bunch! When dealing with futures inside an Actor, it is often helpful to use the pipeTo pattern, which allows actors to send the results of futures to other actors (including itself). To visualize the messages being sent between the two lovers, we have the actors print out text that represent the messages they are sending as if from a play script.

We have Juliet start the conversation.

```scala
class Romeo extends Actor {
  implicit val timeout = Timeout(5 seconds)

  def receive: Receive = {
    case FallInLove(lover) =>
      context.become(loving(lover))
  }

  def loving(lover: ActorRef): Receive = {
    case DoYouLoveMe =>
      println(
        """ROMEO
          |I love you Juliet
          |""".stripMargin)
      sender() ! ILoveYou

      context.system.scheduler.scheduleOnce(1 second) {
        println(
          """ROMEO
            |Do you love me Juliet?
            |""".stripMargin)
        (lover ? DoYouLoveMe).pipeTo(self)
      }
    case ILoveYou =>
      println(
        """ROMEO
          |Yay Juliet loves me
          |""".stripMargin)
  }
}

class Juliet(friar: ActorRef) extends Actor {
  implicit val timeout = Timeout(5 seconds)

  def receive: Receive = {
    case FallInLove(lover) =>
      context.become(loving(lover))
      (lover ? DoYouLoveMe).pipeTo(self)
  }

  def loving(lover: ActorRef): Receive = {
    case DoYouLoveMe =>
      println(
        """JULIET
          |I love you Romeo
          |""".stripMargin)
      lover ! ILoveYou

      context.system.scheduler.scheduleOnce(1 second) {
        println(
          """JULIET
            |Do you love me Romeo?
            |""".stripMargin)
        (lover ? DoYouLoveMe).pipeTo(self)
      }

    case ILoveYou =>
      println(
        """JULIET
          |Yay Romeo loves me
          |""".stripMargin)
  }
}
```

We can run this, and we see the neverending conversation we expect.

```
ROMEO
I love you Juliet

JULIET
Yay Romeo loves me

ROMEO
Do you love me Juliet?

JULIET
I love you Romeo

ROMEO
Yay Juliet loves me

JULIET
Do you love me Romeo?

ROMEO
I love you Juliet

JULIET
Yay Romeo loves me

...
```

Things seem to be going well... for now.

# The Tragedy
Now comes the interesting part, where the happiness of the love of Romeo and Juliet turns into tragedy. This starts when Juliet's parents tell her that she has to marry Paris. We create the message `YouHaveToMarry`, which takes an ActorRef that the recipient must marry. As Capulet (Juliet's Father) doesn't have any role in the tragedy other than telling her to marry Paris, we send the message to Juliet from the main class. We create an Actor class for Paris, whose ActorRef will be sent to Juliet. When Juliet receives the message, she ignores if it is about her lover, otherwise she goes to Friar Lawrence for help (with `HelpMe`).

```scala
object Main extends App {
  val tragedy = ActorSystem("tragedy")
  ...

  tragedy.scheduler.scheduleOnce(5 seconds) {
    println(
      """CAPULET
        |How, how, how, how? Chopped logic! What is this?
        |“Proud,” and “I thank you,” and “I thank you not,”
        |And yet “not proud”? Mistress minion you,
        |Thank me no thankings, nor proud me no prouds,
        |But fettle your fine joints 'gainst Thursday next
        |To go with Paris to Saint Peter’s Church,
        |Or I will drag thee on a hurdle thither.
        |Out, you green sickness, carrion! Out, you baggage!
        |You tallow face!
        |""".stripMargin)
    juliet ! YouHaveToMarry(paris)
  }
}

case class YouHaveToMarry(that: ActorRef)

case object HelpMe

class Juliet(friar: ActorRef) extends Actor {
  ...

  def loving(lover: ActorRef): Receive = {
    case YouHaveToMarry(toMarry) =>
      if (toMarry != lover) {
        println( """JULIET
                   |O, shut the door! And when thou hast done so,
                   |Come weep with me, past hope, past cure, past help.
                   |""".stripMargin)
        friar ! HelpMe
      }
  }
}

class Paris extends Actor {
  def receive: Receive = {
    case _ =>
  }
}
```

No wonder Juliet didn't want to marry Paris, he's so passive!

In the story, when Juliet goes to the Friar for help, he comes up with a plan to let Juliet run away with Romeo. He gives Juliet a sleeping potion, which will make her appear to be dead. After her funeral, Romeo would come and run away with her. We create a message `SleepingPotion`, which the Friar sends to Juliet. When Juliet receives the potion, she enters the `sleeping` state, in which she ignores any incoming messages. In our actor world, Juliet wakes up after 10 seconds by entering the `loving` state again. After waking up, Juliet attempts to restart the conversation with Romeo.

```scala
class FriarLawrence extends Actor {
  def receive: Receive = {
    case HelpMe =>
      println("""FRIAR LAWRENCE
                |Take thou this vial, being then in bed,
                |And this distillèd liquor drink thou off,
                |When presently through all thy veins shall run
                |A cold and drowsy humor, for no pulse
                |Shall keep his native progress, but surcease.
                |""".stripMargin)
      sender() ! SleepingPotion
  }
}

class Juliet(friar: ActorRef) extends Actor {
  ...

  def loving(lover: ActorRef): Receive = {
    ...

    case SleepingPotion =>
      println(
        """JULIET
          |Romeo, Romeo, Romeo! Here’s drink. I drink to thee.
          |""".stripMargin)
      context.become(sleeping)
      context.system.scheduler.scheduleOnce(10 second) {
        context.become(loving(lover))
        println("""JULIET
                  |O comfortable Friar! Where is my lord?
                  |I do remember well where I should be,
                  |And there I am. Where is my Romeo?
                  |""".stripMargin)
        (lover ? DoYouLoveMe).pipeTo(self)
      }
  }

  def sleeping: Receive = {
    case _ =>
  }
}
```

We can now see Friar Lawrence's plan in action:

```
CAPULET
How, how, how, how? Chopped logic! What is this?
“Proud,” and “I thank you,” and “I thank you not,”
And yet “not proud”? Mistress minion you,
Thank me no thankings, nor proud me no prouds,
But fettle your fine joints 'gainst Thursday next
To go with Paris to Saint Peter’s Church,
Or I will drag thee on a hurdle thither.
Out, you green sickness, carrion! Out, you baggage!
You tallow face!

JULIET
O, shut the door! And when thou hast done so,
Come weep with me, past hope, past cure, past help.

FRIAR LAWRENCE
Take thou this vial, being then in bed,
And this distillèd liquor drink thou off,
When presently through all thy veins shall run
A cold and drowsy humor, for no pulse
Shall keep his native progress, but surcease.

JULIET
Romeo, Romeo, Romeo! Here’s drink. I drink to thee.
```

But alas! While Juliet was sleeping Romeo runs into a timeout when his message to her goes unanswered. He assumes that Juliet is dead, and kills himself.

```scala
class Romeo extends Actor {
  ...

  def loving(lover: ActorRef): Receive = {
    ...

    case Status.Failure(e) =>
      println(e)
      println(
        """ROMEO
          |Here’s to my love! (drinks the poison) O true apothecary,
          |Thy drugs are quick. Thus with a kiss I die.
          |""".stripMargin)
      throw new IllegalStateException(
        """ROMEO
          |I cannot live without my lover
          |""".stripMargin)
  }
}
```

And Juliet runs into a timeout when she wakes up, so she kills herself.

```scala
class Juliet(friar: ActorRef) extends Actor {
  ...

  def loving(lover: ActorRef): Receive = {
    ...

    case Status.Failure(e) =>
      println(e)
      println(
        """JULIET
          |O happy dagger,
          |This is thy sheath. There rust and let me die.
          |(stabs herself with ROMEO’s dagger and dies)
          |""".stripMargin)
      throw new IllegalStateException(
        """JULIET
          |I cannot live without my lover
          |""".stripMargin)
}
```

So this is how the plan actually unfolds:

```
ROMEO
Do you love me Juliet?

akka.pattern.AskTimeoutException: Ask timed out on [Actor[akka://tragedy/user/$c#205689067]] after [5000 ms]
ROMEO
Here’s to my love! (drinks the poison) O true apothecary,
Thy drugs are quick. Thus with a kiss I die.

[ERROR] [07/10/2015 14:16:01.876] [tragedy-akka.actor.default-dispatcher-4] [akka://tragedy/user/$a] ROMEO
I cannot live without my lover

java.lang.IllegalStateException: ROMEO
I cannot live without my lover

	at me.shadaj.romeojuliet.Romeo$$anonfun$loving$1.applyOrElse(Tragedy.scala:60)
	at akka.actor.Actor$class.aroundReceive(Actor.scala:465)
	at me.shadaj.romeojuliet.Romeo.aroundReceive(Tragedy.scala:25)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:516)
	at akka.actor.ActorCell.invoke(ActorCell.scala:487)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:254)
	at akka.dispatch.Mailbox.run(Mailbox.scala:221)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:231)
	at scala.concurrent.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at scala.concurrent.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at scala.concurrent.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at scala.concurrent.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

JULIET
O comfortable Friar! Where is my lord?
I do remember well where I should be,
And there I am. Where is my Romeo?

akka.pattern.AskTimeoutException: Ask timed out on [Actor[akka://tragedy/user/$a#-1846106934]] after [5000 ms]
JULIET
O happy dagger,
This is thy sheath. There rust and let me die.
(stabs herself with ROMEO’s dagger and dies)

[ERROR] [07/10/2015 14:16:15.887] [tragedy-akka.actor.default-dispatcher-3] [akka://tragedy/user/$c] JULIET
I cannot live without my lover

java.lang.IllegalStateException: JULIET
I cannot live without my lover

	at me.shadaj.romeojuliet.Juliet$$anonfun$loving$2.applyOrElse(Tragedy.scala:128)
	at akka.actor.Actor$class.aroundReceive(Actor.scala:465)
	at me.shadaj.romeojuliet.Juliet.aroundReceive(Tragedy.scala:67)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:516)
	at akka.actor.ActorCell.invoke(ActorCell.scala:487)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:254)
	at akka.dispatch.Mailbox.run(Mailbox.scala:221)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:231)
	at scala.concurrent.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at scala.concurrent.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at scala.concurrent.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at scala.concurrent.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)
```

Quoting the last line of *Romeo and Juliet*:

>For never was a story of more woe

>Than this of Juliet and her Romeo.

I have open-sourced the code in this blog on [GitHub](http://github.com/shadaj/romeo-juliet).
