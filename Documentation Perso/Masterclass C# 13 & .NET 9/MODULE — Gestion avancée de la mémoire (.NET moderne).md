
# C# 13 & .NET 9 — Cours Complet Exhaustif (Mode Entretien & Expertise)

---
# MODULE — Gestion avancée de la mémoire (.NET moderne)

## 1️⃣ Rappels fondamentaux

### Stack vs Heap
- Stack : mémoire rapide, allocation automatique, durée de vie courte.
- Heap : mémoire managée par le Garbage Collector.

Types valeur → généralement Stack  
Types référence → Heap  

Attention : dépend du contexte (boxing, async, closures).

---
## 2️⃣ Garbage Collector (GC)

- Générations 0, 1, 2
- Compactage mémoire
- Finalizers coûteux
- IDisposable pour ressources externes

Bonnes pratiques :
- Limiter allocations
- Utiliser using
- Pool d’objets si forte pression mémoire

---
## 3️⃣ Native Ints (C# 9.0)

C# 9 introduit :
- nint
- nuint

Entiers dépendants de l’architecture (32/64 bits).

Usage :
- Interop natif
- Manipulation bas niveau
- Code unsafe

Exemple :
nint ptrSize = IntPtr.Size;

À utiliser uniquement si nécessaire.

---
## 4️⃣ Span<T> et Memory<T>

### Span<T>
Structure permettant de manipuler des blocs mémoire sans allocation.

- Stack-only
- Très performant
- Évite copies inutiles

Exemple :
Span<int> numbers = stackalloc int[3] {1,2,3};

Caractéristiques :
- Non stockable sur heap
- Non utilisable directement en async

### Memory<T>
Version heap-safe de Span.

---
## 5️⃣ stackalloc

Allocation sur la stack :

Span<byte> buffer = stackalloc byte[256];

Avantages :
- Ultra rapide
- Aucune pression GC

Risque :
- Stack overflow si allocation trop grande

---
## 6️⃣ Code unsafe

Permet :
- Pointeurs
- Manipulation directe mémoire
- Interop C/C++

Activation :
<AllowUnsafeBlocks>true</AllowUnsafeBlocks>

Exemple :
unsafe
{
    int value = 10;
    int* ptr = &value;
}

À réserver aux cas critiques.

---
## 7️⃣ Bonnes pratiques mémoire (niveau senior)

✔ Minimiser allocations  
✔ Préférer Span en traitement intensif  
✔ Éviter LINQ en hot path critique  
✔ Utiliser ArrayPool<T>  
✔ Profiler les allocations  

---
# MEMENTO ENTRETIEN

- Différence stack / heap ?
- Fonctionnement GC ?
- Pourquoi Span est struct ?
- Différence Span vs Memory ?
- Quand utiliser nint ?
- Risques du unsafe ?

---
# Conclusion

La gestion mémoire en .NET moderne permet haute performance et optimisation fine,
mais nécessite compréhension avancée du runtime.
