# Espacio Vectorial 

Un espacio vectorial, es una cuaterna $V = (\mathbb{V}, +, K, \cdot )$ donde:

- $\mathbb{V}$ es un conjunto no vacío cuyos elementos son llamados **vectores**.
- $K$ es un conjunto de escalares. En este caso, $K = \mathbb{R}$ (números reales).
- $(+)$ es una operación interna llamada **suma de vectores**. Se dice operación interna porque los elementos de entrada y salida pertenecen a $\mathbb{V}$.

$$+ : \mathbb{V} \times \mathbb{V} \rightarrow \mathbb{V}$$

- $(\cdot)$ es una operación llamada **multiplicación por un escalar** (outer operation). Los elementos de entrada no pertenecen al mismo conjunto, pero el resultado sí.

$$\cdot : \mathbb{R} \times \mathbb{V} \rightarrow \mathbb{V}$$

Para que $V$ sea un espacio vectorial, debe cumplir las siguientes propiedades:

1. $(\mathbb{V}, +)$ es un grupo abeliano.

2. **Distribución** del producto escalar respecto a la suma de vectores: \
    Para todo $a \in \mathbb{R}$ y para todo $u, v \in \mathbb{V}$, se cumple que $a
    \cdot (u + v) = a \cdot u + a \cdot v$. 

3. **Asociatividad** del producto escalar respecto a la multiplicación de escalares: Para todo
   $a, b \in \mathbb{R}$ y para todo $v \in \mathbb{V}$, se cumple que $(a \cdot b) \cdot v = a
   \cdot (b \cdot v)$.

4. **Elemento neutro de la suma**: Existe un elemento $0 \in \mathbb{V}$ tal que para
   todo $v \in \mathbb{V}$, se cumple que $v + 0 = 0 + v = v$. Este elemento es llamado
   **vector cero**.
