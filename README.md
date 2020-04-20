# CTFd Boolean Prerequisite

Sources de CTFd : https://github.com/CTFd/CTFd

Fonctionne sur la version 2.3.3

## Présentation

<u>Le cahier des charges est le suivant :</u>

- Pouvoir offrir deux chemins possible au joueur, lorsqu'il en choisit un l'autre ce referme.

CTFd propose de base une fonction prérequis qui permet d'afficher un challenge uniquement si un ou plusieurs challenges ont déjà été résolu.

Pour cela il y a une entrée `requirement` dans la base de données qui contient la liste des challenges qui doivent être résolu par l'utilisateur pour que le challenge soit visible.

Les challenges sont identifié par un ID unique que l'on appellera **chall_id**

Par exemple :

```mysql
id           | 5
...
requirements | {"prerequisites": [1, 2, 8]}
```

Signifie que l'utilisateur doit avoir résolu les challenges 1, 2 et 8 pour pouvoir voir le challenge 5.

L'idée est d'ajouter aux prérequis des **challenges non validés**. Ainsi la condition d'accès à un challenge pourra être qu'un challenge n'est pas été résolu. On pourra donc faire des routes.

## Choix technique

#### Dans le code source

Actuellement le tableau `prerequisites` est remplis de la manière suivante :

**CTFd/themes/admin/assets/js/challenges/requirements.js**

```javascript
export function addRequirement(event) {
  const requirements = $("#prerequisite-add-form").serializeJSON();
[...]
  CHALLENGE_REQUIREMENTS.prerequisites.push(
    parseInt(requirements["prerequisite"])
  );
```

Avec `"#prerequisite-add-form"` qui vient de la page HTML suivante :

**CTFd/themes/admin/templates/modals/challenges/requirements.html**

```html
[...]
<form id="prerequisite-add-form">
	<div class="form-group">
		<select class="form-control custom-select" name="prerequisite">
			<option value=""> -- </option>
			{% for challenge_id in challenges %}
				{% if challenge.requirements %}
					{% if challenge_id not in challenge.requirements['prerequisites'] %}
						<option value="{{ challenge_id }}">{{ challenges[challenge_id] }}</option>
					{% endif %}
				{% elif challenge_id != challenge.id %}
					<option value="{{ challenge_id }}">{{ challenges[challenge_id] }}</option>
				{% endif %}
			{% endfor %}
		</select>
	</div>
	<div class="form-group">
		<button class="btn btn-success float-right">Add Prerequisite</button>
	</div>
</form>

<script>
	var CHALLENGE_REQUIREMENTS = {{ challenge.requirements | tojson }} || {prerequisites: []};
</script>
```

Voilà de ce qui est du remplissage du tableau `prerequisites` dans la base de données.

Pour ce qui est de la vérification des prérequis en backend elle se fait avec ce bout de code Python :

**CTFd/api/v1/challenges.py**

```python
solve_ids = (
    Solves.query.with_entities(Solves.challenge_id)
    .filter_by(account_id=user.account_id)
    .order_by(Solves.challenge_id.asc())
    .all()
)
solve_ids = set([value for value, in solve_ids])  
requirements = challenge.requirements.get("prerequisites", [])
prereqs = set(requirements)    

if solve_ids >= prereqs:
	pass
```

Le fonctionnement est basé sur les **sets** python, documentation [ici](https://snakify.org/fr/lessons/sets/) cela revient à des manipulations sur des ensembles mathématique.

J'ai fait le choix de remplacer le tableau de chall_id `prerequisites` par un tableau de tableaux `prerequisites`.

La structure sera la suivante :

```mysql
{"prerequisites": [[state, chall_id], [state, chall_id], ...]}
```

Ou **state** sera :

- 1 si le challenge ne doit pas être résolu

- 2 si le challenge doit être résolu

<u>**Il va donc falloir :**</u>

- Proposer de choisir la valeur de **state** sur la page HTML
- Continuer d'afficher le nom du challenge (mais pas l'état) sur la page HTML
- Envoyer la nouvelle structure de tableau dans la base de données avec du JavaScript
- Changer la vérification actuelle des prérequis en backend avec du Python.



## Modifications

### Proposer de choisir la valeur de **state** sur la page HTML

Pour cela on va ajouter une liste déroulante qui remplira l'item `state` du tableau `challenge.requirements`.

<u>Voici le code ajouté :</u>

**CTFd/themes/admin/templates/modals/challenges/requirements.html**

```html
		<select class="form-control custom-select" name="state">
			<option value="2">Validé</option>
			<option value="1">Non Validé</option>
		</select>
```

### Continuer d'afficher le nom du challenge (mais pas l'état) sur la page HTML

Pour cela on modifier la manière dont la page trouve la liste des prérequis. On ne va plus chercher le n-ième élément du tableau `requirements['prerequisites']` mais le second élément du n-ième élément du tableau `requirements['prerequisites']`.

<u>Après modification :</u>

**CTFd/themes/admin/templates/modals/challenges/requirements.html**

```html
{% if challenge.requirements %}
	{% for prereq in requirements['prerequisites'] %}
		<tr>
			<td>{{ challenges[prereq[1]] }}</td>
			<td>
				<i role='button' class='btn-fa fas fa-times delete-requirement' challenge-id="{{ prereq }}"></i>
			</td>
		</tr>
	{% endfor %}
{% endif %}
```

Le changement est au niveau de `<td>{{ challenges[prereq[1]] }}</td>`

qui été avant : `<td>{{ challenges[prereq] }}</td>`

### Envoyer la nouvelle structure de tableau dans la base de données avec du JavaScript

On modifie le code présenté au début de la manière suivante pour envoyer non pas un tableau mais un tableau de tableaux à la base de donnée.

```javascript
CHALLENGE_REQUIREMENTS.prerequisites.push(
  //parseInt(requirements["prerequisite"])
  [parseInt(requirements["state"]), parseInt(requirements["prerequisite"])]
);
```

### Changer la vérification actuelle des prérequis en backend avec du Python

<u>Procédure générale :</u>

- On prend en entrée un tableau de prérequis que l'on coupe en deux sous-tableaux,
un tableau des challenges devant être résolu, un pour les challenges ne devant
pas être résolu.

- Pour vérifier que les challenges qui de doivent pas être résolu ne le soit pas,
  on fait l'intersection entre les challenges résolu et les challenges qui ne
  doivent pas être résolu. Cela doit retourner une liste vide.

- Pour vérifier que les challenges qui doivent être résolu le soit on compare
  l'ensemble des challenges résolu à l'ensemble des challenges devant être résolu.
  L'ensemble de ceux résolu doit être supérieur ou égal à ceux devant être résolu.

**Il y aura donc :**

- Une fonction pour vérifier que les challenges ne devant pas être résolu par l'utilisateur ne le soit pas.
- Une fonction pour vérifier que les challenges devant être résolu par l'utilisateur le soit.
- Une fonction qui retourne `True ` si les deux tests précédant sont validé `False` sinon.
- Une fonction qui coupe le tableau en deux et effectue le test global et retourne la valeur de retour du test global.

<u>Le code :</u>

```python
def down_test(solved, down):
    if (list(set(solved).intersection(set(down)))) == []:
        return True
    else:
        return False

def up_test(solved, up):
    return (set(solved)>=set(up))

def global_test(solved, up ,down):
    if up_test(solved, up) and down_test(solved, down):
        return True
    else:
        return False

def compare(required,solved):
    down=[]
    up=[]
    for elem in required:
        if elem[0]==1:
            down.append(elem[1])
        else:
            up.append(elem[1])
    return(global_test(solved,up,down))
```

J'ai ajouté ces 4 définitions en tête du fichier **CTFd/api/v1/challenges.py**.

Au niveau des tests il y en a trois de différents :

- Lors de l'affichage des challenges (L. 42)
- Lors d'une tentative d'accès à la page (L. 148)
- Lors de l'envoi d'un flag pour un challenge (L. 293)

Je ne vais donner que la forme générale.

**<u>Voici le code modifié :</u>**

```python
solve_ids = (
    Solves.query.with_entities(Solves.challenge_id)
    .filter_by(account_id=user.account_id)
    .order_by(Solves.challenge_id.asc())
    .all()
)
requirements = challenge.requirements.get("prerequisites", [])

if compare(requirements,solve_ids):
	pass
```



Tout est bon, vous trouverez tous les fichiers modifiés dans le dossier `patch`.

