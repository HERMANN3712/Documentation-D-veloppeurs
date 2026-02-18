# 🐚 Bash -- Mémento Commandes, Variables & Boucles

> Cheat sheet rapide pour scripting Linux / DevOps / CI-CD

------------------------------------------------------------------------

# 1️⃣ Variables

## 🔹 Déclaration

``` bash
name="Hermann"
age=30
```

⚠️ Pas d'espace autour du `=`

------------------------------------------------------------------------

## 🔹 Utilisation

``` bash
echo $name
echo ${name}
```

Toujours privilégier :

``` bash
echo "Bonjour ${name}Dev"
```

------------------------------------------------------------------------

## 🔹 Lecture utilisateur

``` bash
read username
read -p "Nom: " username
read -s password   # mode silencieux
```

------------------------------------------------------------------------

## 🔹 Variables spéciales

  Variable     Description
  ------------ --------------------
  `$0`         Nom du script
  `$1`, `$2`   Arguments
  `$#`         Nombre d'arguments
  `$@`         Tous les arguments
  `$?`         Code retour
  `$$`         PID du script

------------------------------------------------------------------------

## 🔹 Variables d'environnement

``` bash
export ENV=prod
echo $HOME
echo $PATH
```

------------------------------------------------------------------------

## 🔹 Constante

``` bash
readonly PI=3.14
```

------------------------------------------------------------------------

# 2️⃣ Conditions

## 🔹 If

``` bash
if [ "$age" -gt 18 ]; then
  echo "Majeur"
fi
```

------------------------------------------------------------------------

## 🔹 Comparaisons numériques

  Opérateur   Signification
  ----------- -------------------
  `-eq`       égal
  `-ne`       différent
  `-lt`       inférieur
  `-le`       inférieur ou égal
  `-gt`       supérieur
  `-ge`       supérieur ou égal

------------------------------------------------------------------------

# 3️⃣ Boucles

## 🔹 For (range)

``` bash
for i in {1..5}
do
  echo $i
done
```

------------------------------------------------------------------------

## 🔹 While

``` bash
count=0

while [ $count -lt 5 ]
do
  echo $count
  ((count++))
done
```

------------------------------------------------------------------------

# 4️⃣ Fonctions

``` bash
greet() {
  echo "Hello $1"
}

greet "Hermann"
```

------------------------------------------------------------------------

# 5️⃣ Arithmétique

``` bash
result=$((5 + 3))
echo $result
```

------------------------------------------------------------------------

# 6️⃣ Tableaux

``` bash
arr=("dev" "ops" "cloud")

echo ${arr[0]}
echo ${arr[@]}
echo ${#arr[@]}
```

------------------------------------------------------------------------

# 7️⃣ Redirections

``` bash
ls > out.txt
ls >> out.txt
ls 2> error.txt
ls &> all.txt
```

------------------------------------------------------------------------

# 🔟 Structure propre d'un script

``` bash
#!/bin/bash
set -euo pipefail

main() {
  echo "Script démarré"
}

main "$@"
```

------------------------------------------------------------------------

✅ Prêt pour GitHub\
💡 Compatible Linux / MacOS / WSL
