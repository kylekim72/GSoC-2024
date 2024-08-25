# Support the generation of violation witness in Symbolic PathFinder

## Abstract
Our final goal is to improve the score of Symbolic PathFinder(SPF) in SV-COMP(Software Verification Competition). There are two main reasons why SPF is losing many scores: unconfirmed false verdicts and incorrect results. In this project, we extend SPF to generate a violation witness for all unsafe properties. This support, in turn, improves SPF’s score as these unsafe properties can be confirmed with the generated witness. During the journey of GSoC, I investigated witness tools supported by SV-COMP that check the violation witness. We supported the generation of witnesses in the standardized GraphML form. Using this standard format allows our witnesses to be checked by any witness checker that supports this format. I wrote a code to generate violation witnesses in SPF and ran all the benchmarks at SV-COMP to measure the performance of SPF. Finally, I removed about 80% of unconfirmed false, which is expected to improve SPF’s score from 182 to 360+. 


## Introduction
This section provides a motivation of this project, a brief introduction of witness of the source code, witness validation tools and GraphML format.The picture below shows the result of SPF at last year’s SV-COMP.

<br/><img width="1280" alt="Figure 1" src="https://github.com/user-attachments/assets/fe8735b4-4993-4144-a947-7ee0e00a6d35">

Figure 1 : Statistics of SPF in last year’s SV-COMP


<br/>The first row’s “raw score” column shows SPF’s final score in SV-COMP, 182 points. Below this row, 310 points are the score obtained by “correct results”, especially “correct true”. However, there is 0 score for correct false. Although SPF outputs unsafe when expected verdict is false, it can’t get points since SPF does not generate violation witness that should be validated by witness validators.Therefore, this “correct false” result cannot be asserted surely. In addition, there is a penalized score due to incorrect results, but we will focus on eliminating unconfirmed false results in this project.


<br/><br/>In SV-COMP, violation witness is written in GraphML file format. There are three parts that construct violation witness. First, there is a node part that represents the node of the violation witness. Second, there is an edge part. The edge part contains significant information that could be a evidence of violation, such as a value of certain nondeterministic variable. Finally, there is a header part that declares some attributes that we want to use in violation witness. For example, if you want to specify thread id in your edge, you have to declare at the top of the GraphML file like this : 

```
<key attr.name="threadId" attr.type="int" for="edge" id="threadId">
    <default>0</default>
  </key>
```

Figure 2 : Example of key attribute in GraphML file

<br/>Declaration of attributes is similar to import statement on Java or #include statement in C. For instance, if you want to use printf( ) function in your C source code, you should write “#include <stdio.h>” in your source file.
Here is an example that will improve your understanding of violation witness. Let's assume we want to verify if an assertion error could occur when we run the program below.

```Java
import org.sosy_lab.sv_benchmarks.Verifier;

class Main {
  public static void main(String[] args) {
    int i = Verifier.nondetInt();

    if (i >= 1000) assert i > 1000 : "i is greater 1000"; // should fail

  }
}
```


