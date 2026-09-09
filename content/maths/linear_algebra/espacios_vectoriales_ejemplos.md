---
title: Espacios vectoriales - Ejemplos
draft: true
tags:
  - matematicas
  - algebra-lineal
---

### **Ejemplo 1:** Espacio vectorial $(\mathbb{R}^2, +, \mathbb{R}, ·)$


El espacio esta conformado por:

- $\mathbb{R}^2 = \{(x_1, x_2) | x_1, x_2 \in \mathbb{R}\}$
- $+$: Suma de vectores definida como: 


  $$u = (u_1, u_2), v = (v_1, v_2) \rightarrow u + v = (u_1 + v_1, u_2 + v_2)$$

- $\mathbb{R}$: Conjunto de escalares (números reales)

- $(·)$: Multiplicación por un escalar definida como:
    
    $$a \in \mathbb{R}, v = (v_1, v_2) \rightarrow a · v = (a · v_1, a · v_2)$$


#### 1. Probar que $(\mathbb{R}^2, +)$ es un grupo abeliano.

*Clausura*:

$u = (u_1, u_2), v = (v_1, v_2) \rightarrow u + v = (u_1 + v_1, u_2 + v_2)$

Como $u_1, u_2, v_1, v_2 \in \mathbb{R} \Rightarrow u_1 + v_1, u_2 + v_2 \in \mathbb{R}$

Entonces $u + v \in \mathbb{R}^2$

*Asociatividad*:

Se tienen: $u = (u_1, u_2), v = (v_1, v_2), w = (w_1, w_2) \in \mathbb{R}^2$

Se debe verificar: $(u + v) + w = u + (v + w)$

Operando:

$$
\begin{align}
(u + v) + w =& (u_1 + v_1, u_2 + v_2) + (w_1, w_2)\\
=& ((u_1 + v_1) + w_1, (u_2 + v_2) + w_2) \\
=& (u_1 + (v_1 + w_1), u_2 + (v_2 + w_2)) \\
=& (u_1, u_2) + (v_1 + w_1, v_2 + w_2) \\
=& u + (v + w)
\end{align}
$$


*Elemento Identidad de Suma*

Se tiene $u = (u_1, u_2) \in \mathbb{R}^2$

Se debe encontrar $e \in \mathbb{R}^2$ tal que $u + e = e + u = u$

Operando:

$$
\begin{align}
u + e &= (u_1, u_2) + (e_1, e_2) \\
      &= (u_1 + e_1,\, u_2 + e_2) \\
      &= (u_1, u_2) \qquad \text{si } e_1 = e_2 = 0
\end{align}
$$

Entonces, el elemento identidad es $e = (0, 0)$

*Elemento Inverso de Suma*

Se tiene $u = (u_1, u_2) \in \mathbb{R}^2$

Se debe encontrar $v = (v_1, v_2) \in \mathbb{R}^2$ tal que $u + v = v + u = e = (0, 0)$

Operando:

$$
\begin{align}
u + v &= (u_1, u_2) + (v_1, v_2) \\
      &= (u_1 + v_1,\, u_2 + v_2) \\
      &= (0, 0) \qquad \text{si } v_1 = -u_1 \text{ y } v_2 = -u_2
\end{align}
$$

Entonces, el elemento inverso es $v = (-u_1, -u_2)$. 

*Conmutatividad*:

Se tienen: $u = (u_1, u_2), v = (v_1, v_2) \in \mathbb{R}^2$

Se debe verificar: $u + v = v + u$

Operando:

$$
\begin{align}
u + v &= (u_1, u_2) + (v_1, v_2) \\
      &= (u_1 + v_1, u_2 + v_2) \\
      &= (v_1 + u_1, v_2 + u_2) \qquad \text{conmutat. en } \mathbb{R} \\
      &= v + u
\end{align}
$$







