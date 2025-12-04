# Assignment IV - Discrete Math(H)

**Name**: Yuxuan HOU (侯宇轩)

**Student ID**: 12413104

**Date**: 2025.12.03

## Q.1

![image-20251203204029357](./assets/image-20251203204029357.png)

Base case: Let \(n=1\). Then
\[
2\cdot 1^2+6\cdot 1=2+6=8
\]
Then $4 \mid 8$.

Induction: Assume that for some positive integer \(k\),
\[
4\mid(2k^2+6k).
\]
Hence there exists an integer \(m\) such that
\[
2k^2+6k=4m.
\]

Inductive step:

\[
\begin{aligned}
2(k+1)^2+6(k+1)
&=2(k^2+2k+1)+6k+6 \\
&=2k^2+4k+2+6k+6 \\
&=2k^2+10k+8 \\
&=(2k^2+6k)+4k+8.
\end{aligned}
\]
We have \(2k^2+6k=4m\). Substituting, we obtain
\[
\begin{aligned}
2(k+1)^2+6(k+1)
&=4m+4k+8 \\
&=4m+4(k+2) \\
&=4(m+k+2).
\end{aligned}
\]
Therefore
\[
4\mid\bigl(2(k+1)^2+6(k+1)\bigr),
\]
Hence, for all positive integers \(n\), the integer \(2n^2+6n\) is divisible by \(4\).

## Q.2

![image-20251204221617108](./assets/image-20251204221617108.png)

Left side: For every \(i\), we have \(x\in A_i\), so
\[
x\in A_1\cap A_2\cap\cdots\cap A_n.
\]
At the same time, we also have \(x\notin B\). Hence
\[
x\in (A_1\cap A_2\cap\cdots\cap A_n) -  B.
\]
Therefore,
\[
(A_1- B)\cap\cdots\cap(A_n- B)
\subseteq
(A_1\cap\cdots\cap A_n)- B
\]

Right side:
\[
x\in A_1\cap A_2\cap\cdots\cap A_n
\quad\land\quad
x\notin B.
\]
 \(x\in A_1\cap A_2\cap\cdots\cap A_n\) is equivalent to for each \(i=1,\dots,n\), we have \(x\in A_i\).

Therefore, for every \(i\) we have \(x\in A_i\) and \(x\notin B\), so
\[
x\in A_i\setminus B
\quad\text{for all } i=1,\dots,n.
\]
Using again the definition of intersection, we obtain
\[
x\in (A_1\setminus B)\cap(A_2\setminus B)\cap\cdots\cap(A_n\setminus B).
\]
Hence,
\[
(A_1\cap\cdots\cap A_n)\setminus B
\subseteq
(A_1\setminus B)\cap\cdots\cap(A_n\setminus B).
\]

---

**Conclusion.** Since each side is contained in the other, the two sets
are equal:
\[
(A_1\setminus B)\cap(A_2\setminus B)\cap\cdots\cap(A_n\setminus B)
=
(A_1\cap A_2\cap\cdots\cap A_n)\setminus B.
\]

## Q.3

