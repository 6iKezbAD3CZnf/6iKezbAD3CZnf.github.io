---

author: "6i"
title: "Groth16プロトコルを追う"
date: 2023-01-06T20:37:22+09:00
descritption: "Groth16プロトコルを追う"
tags: []
categories: []
series: []
aliases: []
draft: false
math: true

---

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.16.4/dist/katex.min.css" integrity="sha384-vKruj+a13U8yHIkAyGgK1J3ArTLzrFGBbBc0tDp4ad/EyewESeXE/Iv67Aj8gKZ0" crossorigin="anonymous">
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.4/dist/katex.min.js" integrity="sha384-PwRUT/YqbnEjkZO0zZxNqcxACrXe+j766U2amXcgMg5457rve2Y7I6ZJSm2A0mS4" crossorigin="anonymous"></script>
<script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.4/dist/contrib/auto-render.min.js" integrity="sha384-+VBxd3r6XgURycqtZ117nYw44OOcIax56Z4dCRWbxyPt0Koah1uHoK0o4+/RRE05" crossorigin="anonymous"></script>
<script>
    document.addEventListener("DOMContentLoaded", function() {
        renderMathInElement(document.body, {
          // customised options
          // • auto-render specific keys, e.g.:
          delimiters: [
              {left: '$$', right: '$$', display: true},
              {left: '$', right: '$', display: false},
              {left: '\\(', right: '\\)', display: false},
              {left: '\\[', right: '\\]', display: true}
          ],
          // • rendering keys, e.g.:
          throwOnError : false
        });
    });
</script>

## Groth16とは
この記事では、Groth16の論文[^1]の読んでみての自分なりの理解をまとめます。Groth16とはゼロ知識証明システムの一種で、Jens Grothらが提案されました。ゼロ知識証明とは、あるステートメントが成立することを検証者に対して証明するためのシステムで、かつ成立するということ以外の情報が漏れないという特徴があります。\
この記事では、ある3次方程式の例を用いて証明者が解を知っていることをGroth16プロトコルを用いて検証者に証明する様子を具体的に追っていきます。Groth16の完全性・健全性・ゼロ知識性に関しては触れません。

## Quadratic Arithmetic Program
今回はVitalikさんのこちらの記事[^2]を参考に、以下の3次方程式の解 ($x=3$) を知っていることを解に関する情報を公開することなく、検証者に対して証明したいとします。
$$
    x^3 + x + 5 = 35
$$
Groth16はQuadratic Arithmetic Program (QAP)と呼ばれる形式を対象とするプロトコルなので、この問題をQAPに変換する必要があります。\
QAPは$m$個の$n-1$次方程式$\ v_i(X), u_i(x), w_i(X)\ $と$n$次方程式$\ t(X)\ $に対して、下の恒等式を満たすような$\ \lbrace a_i \rbrace_{i=0}^{m}, h(X)\ $が存在するかという問題です。
$$
    \sum_{i=0}^m a_i v_i(X) \cdot \sum_{i=0}^m a_i u_i(X) = \sum_{i=0}^m a_i w_i(X) + h(X)t(X)
$$

### Flattening
まずは、証明したい方程式$\ x^3 + x + 5 = 35\ $を$\ a\ (+ / \times)\ b = c\ $の形のみで表される形へ平坦化をします。
$$
    x \times x = \text{sym}_1 \newline
    \text{sym}_1 \times x = y \newline
    y + x = \text{sym}_2 \newline
    \text{sym}_2 + 5 = \text{out} \newline
    1 \times 0 = 0 \newline
    \text{out} \times 0 = 0
$$
$\ x^3 + x + 5 = 35\ $を満たす$x$を見つけることは、上の制約式において$\ \text{out}=35\ $のときにすべての制約を満たす$\ \lbrace x, \text{sym}_1, y, \text{sym}_2 \rbrace\ $を見つけることに対応します。

### R1CS
次に平坦化された制約式をRank-1 Constraint System (R1CS)と呼ばれる形式へ変換します。\
R1CSは3つの行列$A, B, C$に対して、下の等式を満たすような各要素が$\ \lbrace 1, \text{out}, x, \text{sym}_1, y, \text{sym}_2 \rbrace \ $に対応する長さ6のベクトル$\bold{a}$が存在するかという問題です。$\odot$は2つのベクトルの各要素の積を取る演算を表します。
$$
    (A \cdot \bold{a}) \odot (B \cdot \bold{a}) = C \cdot \bold{a}
$$
今回の例に対応する行列$A, B, C$は次のようになります。
$$
    A =
        \begin{pmatrix}
            0 & 0 & 1 & 0 & 0 & 0 \newline
            0 & 0 & 0 & 1 & 0 & 0 \newline
            0 & 0 & 1 & 0 & 1 & 0 \newline
            5 & 0 & 0 & 0 & 0 & 1 \newline
            1 & 0 & 0 & 0 & 0 & 0 \newline
            0 & 1 & 0 & 0 & 0 & 0
        \end{pmatrix}
$$
$$
    B =
        \begin{pmatrix}
            0 & 0 & 1 & 0 & 0 & 0 \newline
            0 & 0 & 1 & 0 & 0 & 0 \newline
            1 & 0 & 0 & 0 & 0 & 0 \newline
            1 & 0 & 0 & 0 & 0 & 0 \newline
            0 & 0 & 0 & 0 & 0 & 0 \newline
            0 & 0 & 0 & 0 & 0 & 0
        \end{pmatrix}
$$
$$
    C =
        \begin{pmatrix}
            0 & 0 & 0 & 1 & 0 & 0 \newline
            0 & 0 & 0 & 0 & 1 & 0 \newline
            0 & 0 & 0 & 0 & 0 & 1 \newline
            0 & 1 & 0 & 0 & 0 & 0 \newline
            0 & 0 & 0 & 0 & 0 & 0 \newline
            0 & 0 & 0 & 0 & 0 & 0
        \end{pmatrix}
$$
満たすべき等式の第一要素は次の式と同等になります。
$$
    \begin{pmatrix} 0 & 0 & 1 & 0 & 0 & 0 \end{pmatrix} \cdot \begin{pmatrix} 1 \newline \text{out} \newline x \newline \text{sym}_1 \newline y \newline \text{sym}_2 \end{pmatrix} \times \begin{pmatrix} 0 & 0 & 1 & 0 & 0 & 0 \end{pmatrix} \cdot \begin{pmatrix} 1 \newline \text{out} \newline x \newline \text{sym}_1 \newline y \newline \text{sym}_2 \end{pmatrix} = \begin{pmatrix} 0 & 0 & 0 & 1 & 0 & 0 \end{pmatrix} \cdot \begin{pmatrix} 1 \newline \text{out} \newline x \newline \text{sym}_1 \newline y \newline \text{sym}_2 \end{pmatrix}
$$
$$
    \Leftrightarrow x \times x = \text{sym}_1
$$
このように、平坦化された制約式の解集合とR1CSに変換された問題の解集合が等しいことがわかります。

### QAP
R1CSを別の形で書き換えると次のようになります。
$$
    \tag{$j \in [6]$} \sum_{i=0}^5 a_i A_{ij} \cdot \sum_{i=0}^5 a_i B_{ij} = \sum_{i=0}^5 a_i C_{ij}
$$
QAPの形に近づいてきました。次にLagrange補間を用いて行列の値を複数の多項式に変換します。今後は$\ p = 64513\ $の有限体$\mathbb{F}_p$上の演算を考えます。この有限体から、$\ \omega = 20201\ $を生成源とする位数$8$の巡回群$H$が自然に得られます。\
$n-1$次多項式$\ u_i(X)\ $として

$$
    u_i(\omega^j) = A_{ji}
$$
となるものをLagarange補間により計算すると以下になります。
$$
    \begin{cases}
        u_0(X) &= 42352X^7 + 1895X^6 + 44882X^5 + 32256X^4 + 38289X^3 + 46490X^2 + 35759X + 16129 \newline
        u_1(X) &= 5539X^7 + 36716X^6 + 6045X^5 + 8064X^4 + 58974X^3 + 27797X^2 + 58468X + 56449 \newline
        u_2(X) &= 28652X^7 + 19733X^5 + 48385X^4 + 28652X^3 + 19733X + 48385 \newline
        u_3(X) &= 58974X^7 + 36716X^6 + 58468X^5 + 8064X^4 + 5539X^3 + 27797X^2 + 6045X + 56449 \newline
        u_4(X) &= 36716X^7 + 8064X^6 + 27797X^5 + 56449X^4 + 36716X^3 + 8064X^2 + 27797X + 56449 \newline
        u_5(X) &= 58468X^7 + 27797X^6 + 58974X^5 + 8064X^4 + 6045X^3 + 36716X^2 + 5539X + 56449
    \end{cases}
$$
実際に、$\ u_0(X)\ $の各値は以下のようになります。
$$
    \begin{cases}
    u_0(\omega^0) = 0 \newline
    u_0(\omega^1) = 0 \newline
    u_0(\omega^2) = 0 \newline
    u_0(\omega^3) = 5 \newline
    u_0(\omega^4) = 1 \newline
    u_0(\omega^5) = 0 \newline
    u_0(\omega^6) = 0 \newline
    u_0(\omega^7) = 0
    \end{cases}
$$
同様に$\ v_i(X), w_i(X)\ $は以下のようになります。
$$
    \begin{cases}
        v_0(X) &= 30671X^7 + 35861X^6 + 22258X^5 + 42761X^3 + 44780X^2 + 33336X + 48385 \newline
        v_1(X) &= 0 \newline
        v_2(X) &= 50910X^7 + 28652X^6 + 50404X^5 + 61988X^3 + 19733X^2 + 62494X + 48385 \newline
        v_3(X) &= 0 \newline
        v_4(X) &= 0 \newline
        v_5(X) &= 0
    \end{cases}
$$
$$
    \begin{cases}
        w_0(X) &= 0 \newline
        w_1(X) &= 58468X^7 + 27797X^6 + 58974X^5 + 8064X^4 + 6045X^3 + 36716X^2 + 5539X + 56449 \newline
        w_2(X) &= 0 \newline
        w_3(X) &= 56449X^7 + 56449X^6 + 56449X^5 + 56449X^4 + 56449X^3 + 56449X^2 + 56449X + 56449 \newline
        w_4(X) &= 58974X^7 + 36716X^6 + 58468X^5 + 8064X^4 + 5539X^3 + 27797X^2 + 6045X + 56449 \newline
        w_5(X) &= 36716X^7 + 8064X^6 + 27797X^5 + 56449X^4 + 36716X^3 + 8064X^2 + 27797X + 56449
    \end{cases}
$$
これより、$\ x = \omega^i (i \in [8])\ $において

$$
    \sum_{i=0}^5 a_i v_i(X) \cdot \sum_{i=0}^5 a_i u_i(X) = \sum_{i=0}^5 a_i w_i(X)
$$
が満たす$\ a_i\ $が存在するかどうかという問題に変換されました。\
これは、$\ t(X) = \Pi_{i=1}^8 (X - \omega^i)\ $とおいて、

$$
    \sum_{i=0}^5 a_i v_i(X) \cdot \sum_{i=0}^5 a_i u_i(X) = \sum_{i=0}^5 a_i w_i(X) + h(X)t(X)
$$
が恒等式となる$\ a_i, h(X)\ $が存在するかという問題になり、無事にQAPに変換することができました。\
今回の例である$\ x^3 + x + 5 = 35\ $では

$$
    \begin{align*}
    \bold{a} &= \begin{pmatrix} 1 & 35 & 3 & 9 & 27 & 30 \end{pmatrix} \newline
    h(X) &= 37601X^6 + 56707X^5 + 25690X^4 + 55189X^3 + 20983X^2 + 64096X + 32255
    \end{align*}
$$
が解となります。

## Groth16プロトコル
証明したいステートメントをQAPに変換できましたので、実際にGroth16プロトコルを用いて検証者に対しゼロ知識の証明を行います。\
まず、解である$\ \lbrace 1, \text{out}, x, \text{sym}_1, y, \text{sym}_2 \rbrace \ $のうちどれを公開したくないかを決めます。今回は方程式の解に関する$\ \lbrace x, \text{sym}_1, y, \text{sym}_2 \rbrace \ $にゼロ知識性を持ってほしいので、以下では$\ l=1\ $ ($x$のインデックス-1) と置きます。これらの解をWitnessと呼びます。\
有限体$\mathbb{F}_p$から順回群$\mathbb{G}_1, \mathbb{G}_2$へのエンコードをそれぞれ$[]_1, []_2$で表します。また、$\mathbb{G}_1, \mathbb{G}_2$から$\mathbb{G}_T$へのペアリングを$e()$で表します。

### Setup
セットアップでは信頼できる第三種がランダムに次の値をサンプルし、$\ \alpha, \beta, \gamma, \delta, x \leftarrow \mathbb{Z}_p^*\ $\
今回は$\ \alpha = 10, \beta = 20, \gamma = 30, \delta = 40, x = 50\ $の場合を考えます。\
次によって定義されるCommon Reference String (CRS) $\ \bold{\sigma} = ([\sigma_1]_1, [\sigma_2]_2)\ $を計算し、ProverとVerifierに共有します。

$$
    \begin{align*}
        \bold{\sigma_1} &= \Bigg(
            \begin{array}{cc}
                \alpha, \beta, \delta, \lbrace x^{i} \rbrace_{i=0}^{n-1}, \lbrace \frac{\beta u_i(x) + \alpha v_i{x} + \omega_i(x)}{\gamma} \rbrace_{i=0}^l \newline
                \lbrace \frac{\beta u_i(x) + \alpha v_i{x} + \omega_i(x)}{\delta} \rbrace_{i=l+1}^m, \lbrace \frac{x^it(x)}{\delta} \rbrace_{i=0}^{n-2}
            \end{array}
        \Bigg) \newline
        &= (10, 20, 40, \lbrace 50^i \rbrace_{i=0}^{n-1}, \lbrace 2647, 52647 \rbrace, \lbrace 35780, 55030, 54803, 7884 \rbrace, \lbrace 22029, 4729, 42911, 16621, 56894, 6128, 48348 \rbrace )
    \end{align*}
$$

$$
    \begin{align*}
        \bold{\sigma_2} &= \( \beta, \gamma, \delta, \lbrace x^i \rbrace_{i=0}^{n-1} \) \newline
        &= (20, 30, 40, \lbrace 50^i \rbrace_{i=0}^{n-1})
    \end{align*}
$$

### Prove
Proveでは証明者がランダムに次の値をサンプルし、\
$r, s \leftarrow \mathbb{Z}_p$\
今回は$\ r = 60, s = 70\ $の場合を考えます。\
証拠$\ \pi = ([A]_1, [B]_2, [C]_1)\ $を計算します。

$$
    \begin{align*}
        A &= \alpha + \sum_{i=0}^6 a_i u_i(x) + r \delta \newline
          &= 10 + \sum_{i=0}^6 a_i u_i(50) + 60 \times 40 \newline
          &= 10931
    \end{align*}
$$

$$
    \begin{align*}
        B &= \beta + \sum_{i=0}^6 a_i v_i(x) + s \delta \newline
          &= 20 + \sum_{i=0}^6 a_i v_i(50) + 70 \times 40 \newline
          &= 14001
    \end{align*}
$$

$$
    \begin{align*}
        C &= \frac{\Sigma_{i=l+1}^m a_i \(\beta u_i(x) + \alpha v_i(x) + w_i(x)\) + h(x)t(x)}{\delta} + As + Br - rs \delta \newline
          &= \frac{\Sigma_{i=3}^6 a_i \(20 \times u_i(50) + 10 \times v_i(50) + w_i(50)\) + h(50)t(50)}{40} + 10931 \times 70 + 14001 \times 60 - 60 \times 70 \times 40 \newline
          &= 11622
    \end{align*}
$$

### Verify
検証者は証明者から受け取る証拠$\ \pi = (A^\prime, B^\prime, C^\prime) = ([10931]_1, [14001]_2, [11622]_1)\ $を受け取り以下の等式が成り立つかを検証します。

$$
    \begin{align*}
        &e(A^\prime, B^\prime) = e([\alpha], [\beta]) + e\Bigg( \begin{bmatrix} \sum_{i=0}^l a_i \frac{\beta u_i(x) + \alpha v_i(x) + \omega_i(x)}{\gamma} \end{bmatrix}, [\gamma] \Bigg) + e(C^\prime, [\delta]) \newline
        &\Leftrightarrow e([10931]_1, [14001]_2) = e([10], [20]) + e([38928], [30]) + e([11622], [40]) \newline
        &\Leftrightarrow e([20095], [1]) = e([1], [20095])
    \end{align*}
$$

実際に等式が成り立つことが確認できました。\
以上が具体的なGroth16プロトコルの計算例になります。暗号学的な仮定のもと、最後のVerifyにおける式が成り立つことがステートメントの成立を保証し、かつWitnessに関するゼロ知識性が成り立ちます。

## まとめ
今回はある3次方程式を例に、Groth16プロトコルの具体的な計算を追いました。前半では解を知っていることを証明したい方程式の例をQAPに変換し、後半では論文に従ってプロトコルを計算しました。\
また今回の記事を作成するにあたっては、こちらのレポジトリ[^3]で計算を行いました。

[^1]: [Groth16 Paper](https://link.springer.com/chapter/10.1007/978-3-662-49896-5_11)
[^2]: [Vitalik's Post](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)
[^3]: [learning-bellman-groth16](https://github.com/6iKezbAD3CZnf/learning-bellman-groth16)
