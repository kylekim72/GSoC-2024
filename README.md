# Support the generation of violation witness in Symbolic PathFinder

## Abstract
Our final goal is to improve the score of Symbolic PathFinder(SPF) in SV-COMP(Software Verification Competition). There are two main reasons why SPF is losing many scores: unconfirmed false verdicts and incorrect results. In this project, we extend SPF to generate a violation witness for all unsafe properties. This support, in turn, improves SPF’s score as these unsafe properties can be confirmed with the generated witness. During the journey of GSoC, I investigated witness tools supported by SV-COMP that check the violation witness. We supported the generation of witnesses in the standardized GraphML form. Using this standard format allows our witnesses to be checked by any witness checker that supports this format. I wrote a code to generate violation witnesses in SPF and ran all the benchmarks at SV-COMP to measure the performance of SPF. Finally, I removed about 80% of unconfirmed false, which is expected to improve SPF’s score from 182 to 360+. 


## 1. Introduction
This section provides a motivation of this project, a brief introduction of witness of the source code, witness validation tools and GraphML format.The picture below shows the result of SPF at last year’s SV-COMP.

<br/><img width="1280" alt="Figure 1" src="https://github.com/user-attachments/assets/fe8735b4-4993-4144-a947-7ee0e00a6d35">

Figure 1 : Statistics of SPF in last year’s SV-COMP


<br/>The first row’s “raw score” column shows SPF’s final score in SV-COMP, 182 points. Below this row, 310 points are the score obtained by “correct results”, especially “correct true”. However, there is 0 score for correct false. Although SPF outputs unsafe when expected verdict is false, it can’t get points since SPF does not generate violation witness that should be validated by witness validators.Therefore, this “correct false” result cannot be asserted surely. In addition, there is a penalized score due to incorrect results, but we will focus on eliminating unconfirmed false results in this project.


<br/><br/>In SV-COMP, violation witness is written in GraphML file format. There are three parts that construct violation witness. First, there is a node part that represents the node of the violation witness. Second, there is an edge part. The edge part contains significant information that could be a evidence of violation, such as a value of certain nondeterministic variable. Finally, there is a header part that declares some attributes that we want to use in violation witness. For example, if you want to specify thread id in your edge, you have to declare at the top of the GraphML file like this : 

```html
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

Figure 3 : An example benchmark program that assertion error could be occurred.


<br/>It depends on the value of nondeterministic variable i. If i is greater than 1000, it would pass the assertion statement “i > 1000”. However, if i is 1000, it passes the condition in if statement and it will occur assertion error. The violation witness of the program above looks like below : In node n0, it specifies that node n0 is the start state of the violation witness with key “entry”. In n1, it says that node n1 is the state that contains error with the key “violation”. In edge from n0 to n1, there are 5 keys that show the information of the program above. 
<br/><br/>First, the key “originfile” denotes the filename of the source code, which is “Main.java”. The key “startline” shows the line number of the variable i in the program. The key “threadId” denotes the thread id that executes the program. The key “assumption” represents the value of the variable at corresponding line number at “startline” key. Finally, the key “assumption.scope” denotes the information of which class and method includes the variable. In this case, variable i is in the Main class of the program, and the main( ) method has variable i, thus assumption scope is written as below. Note that there are an essential keys to construct violation witness. For example, the key “threadId” could be deleted if program does not use multithreading. However, the four keys, “originfile”, “startline”, “assumption”, “assumption.scope” must be specified in violation witness. If one of these things is missing, witness validator tools output error.


```html
<node id="n0">
             <data key="entry">true</data>
       </node>

       <node id="n1">
             <data key="violation">true</data>
       </node>
       <edge source="n0" target="n1">
         <data key="originfile">Main.java</data>
         <data key="startline">21</data>
         <data key="assumption">int0 == 1000</data>
         <data key="assumption.scope">java::LMain;</data>
       </edge>
```

Figure 4 : Node and edge part of the violation witness of Figure 3


<br/>There are 2 tools used in SV-COMP to validate violation witnesses, which are wit4java and gwit. Both two tools works similarly, take Java program and violation witness as input, executes the program with matching assumption in violation witness to nondeterministic variable, judge whether violation witness is correct or not. These tools also output “could not validate witness” if it can’t decide whether the violation witness is valid or not. For instance, if you have two nondeterministic variable and you only specify a information of just one nondeterministic variable, witness validator will output “could not validate witness”.


## 2. Implementation Process
Here I’ll give a brief introduction of SPF’s structure and detailed description of how to construct violation witness while SPF is running. Section 2 is divided into five subsections, 2.1 will be a brief introduction to SPF, 2.2 ~ 2.4 will be an introduction to components of violation witness, which are header, edge and node, respectively. 2.5 is an example of SPF’s witness generation and validating this on wit4java.
<br/><br/>

### 2.1 Symbolic PathFinder


SPF is an extension of Java PathFinder that symbolically executes Java bytecode. Like JPF, SPF requires as input : 1. class file, 2. configuration file specifying which methods in the program should be executed symbolically, 3. properties to verify. SPF relies on JPF to systematically explore the different symbolic execution paths.


<br/>In this project, we will focus on SymbolicListener and PathCondition. SymbolicListener listens the bytecode of the input program, and PathCondition contains the information of current state. For example, consider an input program that has conditional statement over two integer variables x and y, like if (x > y). Then consider we are in the state that x is larger than y(i.e. x > y). The PathCondition is (x > y) in current state. If SPF detects violation in the program, SPF outputs PathCondition that causes violation. For instance, if you run the program at Figure 3 on SPF, SPF will output a path condition that shows an integer variable equals 1000, such as “int0[1000] == CONST_1000”.
<br/><br/>

### 2.2 Header


The header of the violation witness consists of declarations of key attributes that will be used to represent the violation witness. For example, if you want to use the “startline” key attribute, you have to declare it at the top of the witness file like Figure 2. To construct the header of the witness, I made a template for witness generation composed of definitions of necessary keys, such as “assumption” or “startline”. When SPF generates the violation witness, the SymbolicListener reads the template and writes the contents of the template to the violation witness.  
<br/>

### 2.3 Edge


As I mentioned at the latter part of the section 1, the required keys to construct a violation witness are “originfile”, “startline”, “assumption” and “assumption.scope”. To obtain these information, I used internal methods of SymbolicListener. 

<br/>For “originfile” and “assumption.scope”, these two information could be simply obtained by invoking methods that related to filename of program and class name, then just parse them out. Before we talk about how to get “startline” and “assumption”, I’ll give you some notable information about SV-COMP. In SV-COMP’s benchmarks, they use `Verifier.nondet~~()` method to use nondeterministic variable. For example, like figure 3, they use `Verifier.nondetInt()` to put a nondeterminisitic value on an integer variable. It means that to get the value of “startline”, we should focus on invocation of `Verifier.nondet~~()`. When the invocation of certain method occurs during execution of SPF, SymbolicListener can get the line number of method by invoking getLineNumber( ). Furthermore, we can also get variable name when SPF generates symbolic value. I created a list that contains the value of these 4 keys and saved the data here. 
<br/><br/>

### 2.4 Node


The techniques that I talked above is the way to construct the edge part of the violation witness. Then how about the node part? It’s pretty simple. Since SV-COMP only uses `Verifier.nondet~~()` method to put nondeterministic value, we just count the number of invocation of `Verifier.nondet~~()`, and that will be the number of the node. For example, if there is a program that has two invocation of `Verifier.nondetInt()` and one invocation of `Verifier.nondetChar()`, the number of nodes of violation witness will be three.
<br/><br/>

### 2.5 Example


For example, let’s say we run program at Figure 3 on SPF. Then SPF will generate a corresponding violation witness, named “witness.graphml”. To run this on wit4java, you need to specify a path to the violation witness, a classpath that is used in the program and a path to the program. In this case, it would be a path to class Verifier and the program. Figure 5 shows a command that wit4java validates the generated witness.

<img width="1280" alt="Figure 5" src="https://github.com/user-attachments/assets/68cc69cb-049c-4cce-8c48-4cff1627e570">

Figure 5 : Validate generated witness by wit4java. Witness file path is specified with `--witness`, classpath and path to the program are specified with `--local-dir`.


## 3. Contribution

In this section, I’ll introduce my contribution to my organization and witness validator tools. 


### 3.1 Supporting Witness Generation


I added the code that generates violation witness in SPF. Check the pull request that I made : https://github.com/SymbolicPathFinder/jpf-symbc/pull/104
Note : This code will not be merged in the deadline(It needs more cleanup). Due to this issue, I’m writing here the latest commit number before the deadline. The last commit number is 1e91da8.
<br/><br/>

### 3.2 Evaluating Witness Generation

I ran all the benchmarks on SPF to measure the performance of SPF. With my witness generation, SPF we fixed 80% of unconfirmed false, which is expected to improve SPF’s score from 182 to 360+. Here is the sheet that I’ve made. Column A denotes the name of the benchmark, Column B denotes the expected verdict of benchmark, which is a ground-truth result. Column C shows the answer of SPF, Column D shows the answer of wit4java, which is one of the violation witness validator. Lastly, Column E represents the score that SPF get by running benchmark at Column A. If you want to filter out the score that I made, set Column B to false, Column C to UNSAFE, and Column D to true. Here is the link : [https://docs.google.com/spreadsheets/d/1m1NGG_h7Q-W1QdvSUO_LcNUoV5S1HvTOrzDdRLsssW4/edit?usp=sharing][https://docs.google.com/spreadsheets/d/1m1NGG_h7Q-W1QdvSUO_LcNUoV5S1HvTOrzDdRLsssW4/edit?usp=sharing]
<br/><br/>

### 3.3 Other Contributions


During the project, there is an issue when I’m writing a code for witness generation. The tool wit4java has different regular expression for parsing the value of assumption(you can see specific code at here : https://github.com/wit4java/wit4java/blob/main/wit4java%2Fprocessors.py#L113, and the full issue that I made is here : https://github.com/wit4java/wit4java/issues/25).
<br/>

This is not a problem when the type of assumption value is numeric type such as int or double. For string type, there is a difference. For other tools except GDart, assumption is represented by using equality operator, such as s = “Hello” . However, for GDart, they use the .equals( ) method to represent string value, for example s.equals(“Hello”). For flexibility of our witness generation, we decided to use .equals( ) method to express string value in assumption when we allow method invocation in violation witness since gwit, which is another violation witness validator, doesn’t allow the equality operator to represent the string value. To achieve this, I added an issue on wit4java’s repository to include the producer name “SPF” at the if statement where GDart is located. Here is the link of the issue : LINK-TO-THE-ISSUE
<br/>


In addition, I found a buggy behavior on wit4java, thus I created an issue on their repository. When wit4java handles string value with equality operator, the space character at the right-hand side causes a problem. For example, let’s assume there is an violation witness that contains string value that represented in assumption such as s=”Hello”, and wit4java accepts this. When I denote the assumption as s = “Hello”, wit4java outputs “could not validate witness”. This is pretty weird, so I made an issue to report this. Here is the link : LINK-TO-THE-ISSUE
<br/>


## 4. Challenges

1. It was my first time seeing a violation witness. To understand the format of the witness, I explored the official GraphML document and other witness tools that participated at SV-COMP to improve my understanding of witness.
<br/>

2. Furthermore, understanding how wit4java works is also a challenging part. This project not only required learning how to use wit4java but also demanded a deep understanding of how wit4java operates. One of the challenging parts was figuring out how the tool parses violation witness files and how it verifies witnesses. To overcome this, I communicated with the wit4java maintainer through GitHub issues, ran the tool multiple times, and even analyzed the wit4java code directly.
<br/>

3. Since SPF was also a tool I used for the first time, understanding its architecture was another challenging aspect. SPF consists of many components, but the most important one for this project was the SymbolicListener. To grasp the structure of SymbolicListener, I read the code, debugged, and ran various examples to deepen my understanding of it. Additionally, I analyzed the bytecode of programs in the benchmark to determine how to capture the necessary information for the witness through SymbolicListener, and then I implemented this.
<br/>

4. Since English is not my first language, there were times when I couldn't fully convey what I wanted to say. Although it was a bit challenging to fully understand the meeting discussions at first, I gradually overcame this by attending multiple meetings. This was possible largely thanks to the consideration and support of my mentors, who were very understanding.
<br/>

5. This was my first time participating in such a large project, so at first, I was a bit nervous, flustered, and somewhat hesitant in my approach. However, thanks to the encouragement from my mentors, I kept evolving throughout the project, and now I feel like I've become someone who communicates actively.



