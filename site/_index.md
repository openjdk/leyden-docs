
Project Leyden
==============

<div style="float: right; padding: 0 0 0 3em;">
  <p>
    <img src="leyden-jar-200.jpg" alt="Leyden jar" width="97" height="200"/>
  </p>
</div>

The primary goal of this Project is to improve the startup time,
time to peak performance, and footprint of Java programs.

<p class="br"> This Project is sponsored by the <a
href="/groups/hotspot/">HotSpot</a> and <a
href="/groups/core-libs/">Core Libraries</a> Groups. </p>


Design notes
------------

  - [Thoughts on Training Runs](https://openjdk.org/projects/leyden/notes/05-training-runs) (2024/9)
  - The "Premain" JEPs: [JEP 483: AOT Class Loading & Linking](https://openjdk.org/jeps/483) (2024/8),<br/>
    [JEP draft: Ahead-of-Time Code Compilation](https://openjdk.org/jeps/8335368)<br/>
    [JEP draft: Ahead-of-Time Method Profiling](https://openjdk.org/jeps/8325147)
  - [Condensing Indy Bootstraps](notes/04-condensing-bootstraps) (2023/8)
  - [Toward Condensers](notes/03-toward-condensers) (2023/7)
  - [Selectively Shifting and Constraining
    Computation](notes/02-shift-and-constrain) (2022/10)
  - [Project Leyden: Beginnings](notes/01-beginnings) (2022/5)

Presentations
-------------

  - _Project Leyden Update_, Ioi Lam, Dan Heidinga,<br/>
    [JVMLS&nbsp;2024](https://openjdk.org/projects/mlvm/jvmlangsummit/)
    ([video](https://youtu.be/OOPSU4LnKg0))
  - _Project Leyden: Capturing Lightning in a Bottle_, Per Minborg,<br/>
    Devoxx&nbsp;UK 2024 ([video](https://youtu.be/teXijm79vno))
  - _Choose Your Own Performance, a Project Leyden Update_,<br/>Dan
    Heidinga, DevNexus&nbsp;2024 ([video](https://youtu.be/NZSbZkKO90Y),
    [slides](slides/leyden-heidinga-devnexus-2024-03.pdf))
  - _Premain Case Study: Spring PetClinic_, Vladimir Ivanov, 2023/9
    ([slides](slides/leyden-premain-petclinic-2023-09-12.pdf))
  - _Project Leyden: Capturing Lightning in a Bottle_, Mark Reinhold,<br/>
    John Rose,
    [JVMLS&nbsp;2023](https://openjdk.org/projects/mlvm/summit2023/)
    ([video](https://youtu.be/lnth19Kf-x0),
     [slides](slides/leyden-jvmls-2023-08-08.pdf))


Resources
---------

  - Mailing list: [leyden-dev](https://mail.openjdk.org/mailman/listinfo/leyden-dev)
    (you must subscribe to the list in order to post to it)
  - Repository: [openjdk/leyden](https://github.com/openjdk/leyden)
  - Early access builds: <https://jdk.java.net/leyden/> (including "Premain" work)

