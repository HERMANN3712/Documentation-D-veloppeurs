# Architecture MVC & MVVM en C#

---

# 🏗️ MVC – Model View Controller

## 📌 Principe
Séparer :
- **Model** → données + logique métier  
- **View** → interface utilisateur  
- **Controller** → reçoit les actions utilisateur et orchestre

## 🔁 Fonctionnement
1. L’utilisateur agit sur la **View**
2. La **View** appelle le **Controller**
3. Le **Controller** modifie le **Model**
4. Le **Model** notifie la **View** (ou le Controller renvoie une nouvelle View)

## 🧠 Résumé entretien
> MVC sépare la responsabilité entre interface, logique métier et gestion des requêtes HTTP.  
> Le Controller est le point d’entrée.

---

# 🧩 Exemple MVC en C# (Web)

## 🎯 Cas : afficher un utilisateur

### 🧠 Model

```csharp
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

---

### 🎮 Controller

```csharp
using Microsoft.AspNetCore.Mvc;

public class UserController : Controller
{
    public IActionResult Details()
    {
        var user = new User
        {
            Name = "Alice",
            Age = 30
        };

        return View(user);
    }
}
```

---

### 🖥️ View (Razor)

```html
@model User

<h2>User Details</h2>
<p>Name: @Model.Name</p>
<p>Age: @Model.Age</p>
```

---

## 🔎 Flux MVC

```
Requête HTTP → Controller → Model → View → HTML
```

👉 Le Controller décide quoi afficher.

---

# 🏗️ MVVM – Model View ViewModel

## 📌 Principe

- **Model** → données + logique métier  
- **View** → interface  
- **ViewModel** → expose des propriétés et commandes pour la View

## 🔁 Différence clé avec MVC

- Pas de Controller  
- La View est liée au ViewModel via **data binding**

---

# 🧩 Exemple MVVM en C# (Desktop)

## 🎯 Cas : afficher et modifier un utilisateur

### 🧠 Model

```csharp
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

---

### 🧠 ViewModel

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

public class UserViewModel : INotifyPropertyChanged
{
    private string _name;
    private int _age;

    public string Name
    {
        get => _name;
        set
        {
            _name = value;
            OnPropertyChanged();
        }
    }

    public int Age
    {
        get => _age;
        set
        {
            _age = value;
            OnPropertyChanged();
        }
    }

    public event PropertyChangedEventHandler PropertyChanged;

    protected void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

---

### 🖥️ View (XAML)

```xml
<StackPanel>
    <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
    <TextBox Text="{Binding Age}" />
    <TextBlock Text="{Binding Name}" />
</StackPanel>
```

---

## 🔎 Flux MVVM

```
View ⇄ ViewModel ⇄ Model
         (Data Binding)
```

👉 La View ne connaît pas le Model  
👉 Pas de Controller  
👉 Tout passe par le ViewModel

---

# 🧩 MVVM propre avec RelayCommand

## 🧠 RelayCommand

```csharp
using System;
using System.Windows.Input;

public class RelayCommand : ICommand
{
    private readonly Action _execute;
    private readonly Func<bool> _canExecute;

    public RelayCommand(Action execute, Func<bool> canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter) => _canExecute?.Invoke() ?? true;
    public void Execute(object parameter) => _execute();
    public event EventHandler CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}
```

---

## 🧠 ViewModel avec Command

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;
using System.Windows.Input;

public class UserViewModel : INotifyPropertyChanged
{
    private string _name;
    public string Name
    {
        get => _name;
        set { _name = value; OnPropertyChanged(); }
    }

    public ICommand SaveCommand { get; }

    public UserViewModel()
    {
        SaveCommand = new RelayCommand(Save, CanSave);
    }

    private void Save()
    {
        Console.WriteLine($"Saved {Name}");
    }

    private bool CanSave() => !string.IsNullOrWhiteSpace(Name);

    public event PropertyChangedEventHandler PropertyChanged;
    private void OnPropertyChanged([CallerMemberName] string prop = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(prop));
}
```

---

### 🖥️ View (XAML)

```xml
<TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
<Button Content="Save" Command="{Binding SaveCommand}" />
```

---

# 🎯 Résumé entretien rapide

**MVC :**
> Architecture orientée requêtes HTTP avec un Controller central.

**MVVM :**
> Architecture orientée interface riche utilisant le data binding et des ICommand pour rendre le ViewModel testable et découplé.

---

# ⚖️ Comparaison rapide

| MVC | MVVM |
|------|------|
| Controller central | Pas de Controller |
| Adapté Web | Adapté Desktop / Mobile |
| Basé sur requêtes | Basé sur Data Binding |
| View dépend du Controller | View liée au ViewModel |

---

# ✅ Points clés

- MVC est courant en développement Web
- MVVM est idéal pour applications Desktop / Mobile
- RelayCommand permet un MVVM propre et testable
- Respect des principes SOLID et séparation des responsabilités

---

Fin du document.

