# Grupo 

Un grupo es un par ordenado $(G, \odot)$, compuesto por un conjunto $G$ y una operación
$\odot$. 

$$\odot := G \times G \rightarrow G$$

Que cumple las siguientes propiedades:

1. **Clausura**: Para todo $a, b \in G$, el resultado de la operación $a \odot b$
   también pertenece a $G$.
2. **Asociatividad**: Para todo $a, b, c \in G$, se cumple que $(a \odot b) \odot c = a$
3. **Elemento identidad**: Existe un elemento $e \in G$ tal que para todo $a \in G$,
   se cumple que $a \odot e = e \odot a = a$.
4. **Elemento inverso**: Para cada elemento $a \in G$, existe un elemento $b \in G$
   tal que $a \odot b = b \odot a = e$, donde $e$ es el elemento identidad.

Además si se cumple la propiedad conmutativa:

5. **Conmutatividad**: Para todo $a, b \in G$, se cumple que $a \odot b = b \odot a$.

Entonces el grupo se denomina **grupo abeliano** o **grupo conmutativo**.
