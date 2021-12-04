# Lab 3 - Load Balancing

> Authors : Jean-Luc Blanc, Dylan Canton, Christian Zaccaria
> Date : 01.12.2021

[TOC]

## Introduction
Nous devons tester une architecture de deux serveurs avec un proxy permettant de faire du load balancing. Il est question de pouvoir tester différents algorithmes de load balancing (en particulier l’algorithme Round-Robin) afin de mieux se familiariser avec la répartition de charge.

> A noter que dans ce laboratoire nous n'avons pas réussi à avoir d'adresses IP sur les containers (merci Windows...), nous avons alors fait le labo en `localhost`. Toutes les captures et IP à insérer dans le rapport seront en fait une adresse `localhost`.
## Task 1 : Install the tools

### Question 1
 > Explain how the load balancer behaves when you open and refresh the URL [http://192.168.42.42](http://192.168.42.42/) in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.

Lorsque nous lançons l'application et essayons d'atteindre l'adresse du proxy (ici `localhost`) on reçoit que la réponse depuis `s1`.

![](media/T1.1-2.PNG)

Lorsqu'on va recharger la page, au contraire on va reçevoir la réponse depuis `s2`. 

![](media/T1.1.PNG)

Cette altérnance va se produire tout le temps. On peut aussi essayer sur une autre fenêtre et avoir le même constat. Nous en déduisons alors que ce comportement n'est pas normal étant donné que nous utilisons des *Cookies* et donc devrions avoir **toujours** le même serveur qui nous repond tant que nous sommes sur la même session (fenêtre). 


### Question 2
> Explain what should be the correct behavior of the load balancer for session management.

Le but d'un *load balancer* bien configuré est de pouvoir garder la même session : et ceci même en rechargeant la page.

Il existe alors 2 possibilités :

- La session est stockée par un des serveurs afin qu'on ne change pas de serveur (même en rechargeant la page).
- La session est transmise entre serveurs, ceci gardant toujours la session active.

### Question 3
> Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2

![](media/T1.3.PNG)

### Question 4
> Provide a screenshot of the summary report from JMeter.

A noter que nous avons du changer l'IP pour effectuer les requêtes (`192.168.42.42 -> localhost`)

![](media/T1.4.PNG)

La capture ci-dessous nous montre que le load balancer réparti de manière équitable les requêtes entre chaque serveur.

![](media/T1.4-2.PNG)

### Question 5
> Run the following command: `docker stop s1` . Clear the results in JMeter and re-run the test plan. Explain what is happening when only one node remains active. Provide another sequence diagram using the same model as the previous one.

![](media/T1.5.PNG)

On remarque que les requêtes sont toutes redirigés vers `s2`. De plus lors de l'arrêt de `s1`, on a remarqué que le load balancer "se bloque" pendant une dixaine de secondes afin de rendre compte que `s1` n'est plus présent. Les requêtes sont ensuite reprises et envoyé sur `s2` comme montré dans la capture ci-dessus.

On constate d'ailleurs que nous ne changeons plus de serveur (on reste sur `s2`) et que la session est bien gardée (voir capture ci-dessous).

![](media/T1.5-2.PNG)

![](media/T1.5-4.PNG)

## Task 2 : Sticky sessions

### Question 1
> There is different way to implement the sticky session. One possibility is to use the SERVERID provided by HAProxy. Another way is to use the NODESESSID provided by the application. Briefly explain the difference between both approaches (provide a sequence diagram with cookies to show the difference).
> Choose one of the both stickiness approach for the next tasks.

`SERVERID`

Cette méthode permet d'injecter un deuxième cookie dans le navigateur du client contenant l'`ID` du serveur qui va traiter la requête. Ceci produit comme résultat que lors d'une prochaine requête, ce cookie indiquera au load balancer sur quel serveur se rendre.

Le nouveau (deuxième) cookie peut ressembler à ceci : `Cookie : SERVERID=s2`

Voici le diagramme de séquence possible :

![](media/T2.1.PNG)



`NODESESSID`

Cette méthode permet d'utiliser la configuration du cookie d'application : on va alors utiliser qu'un seul cookie auquel on prefixe quel serveur traite la requête.

Le cookie peut ressembler à ceci : `Cookie : NODESESSID=s2~<VALEUR_COOKIE>` (`~` : permet de séparer la valeur du cookie et l'information du serveur).

Voici le diagramme de séquence possible :

![](media/T2.1-2.PNG)

### Question 2
> Provide the modified `haproxy.cfg` file with a short explanation of the modifications you did to enable sticky session management.

![](media/T2.2.PNG)

Voici les modifications qui ont été effectué dans le fichier de configuration (dans la partie `backend node`).

Dans le premier encadré on défini le nom du cookie à injecter (dans ce cas `SERVERID`) dans le cas ou un client ne l'ai pas encore. De plus, on va ajouter l'argument `nocache` permettant de ne pas garder les cookies personnels en cache.

Dans le deuxième encadré on va modifier la liste des noeuds en indiquant de vérifier les cookies en choisissant le serveur en fonction du cookie.

### Question 3
> Explain what is the behavior when you open and refresh the URL [http://192.168.42.42](http://192.168.42.42/) in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management. 

Lors du premier accès à l'application on peut vérifier que *HAProxy* à bien ajouté le cookie contenant le serveur qui a répondu.

![](media/T2.3.PNG)

Toutes les prochaines requêtes vont alors utiliser le même serveur tant que l'on ferme pas le navigateur : le cas écheant le cookie sera supprimé et `sessionViews` sera remis à *0*.

On peut voir ici une nouvelle requête (navigateur toujours ouvert depuis le screen précédent). On remarque alors que le `SERVERID` est identique et que `sessionViews` s'est bien incrémenté de *1*.

![](media/T2.3-2.PNG)

### Question 4
>Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. We also want to see what is happening when a second browser is used.

![](media/T2.4.PNG)

### Question 5
> Provide a screenshot of JMeter's summary report. Is there a difference with this run and the run of Task 1?
> - Clear the results in JMeter.
> - Now, update the JMeter script. Go in the HTTP Cookie Manager and verify that the box `Clear cookies each iteration?` is unchecked.
> - Go in `Thread Group` and update the `Number of threads`. Set the value to 2.

![](media/T2.5.PNG)

On ne voit aucun différence dans le `summary report`, néanmoins lorsqu'on regarde sous l'onglet `View Result Tree` on remarque che chaque thread s'exécute sur le même serveur (`s1` sur le thread 1 et `s2` sur le deuxième thread). Dans le cas précédent, on remarquait que le seul thread qu'il y avait s'executait en altérnance sur les deux serveurs.

![](media/T2.5-2.PNG)

![](media/T2.5-3.PNG)

### Question 6

> Provide a screenshot of JMeter's summary report. Give a short explanation of what the load balancer is doing.

![](media/T2.6.PNG)

Le load balancer va équilibrer la charge (les requêtes) en exécutant chaque équitablement sur chaque serveur.

Si il reçoit *X* requête et 1 seconde après il en reçoit *Y* d'une autre cible, il va effectuer les *X* sur un serveur (par exemple `s1`) puis va s'arrêter pour faire celles *Y* (sur `s2`) et ceux en faisant un équilibre.

## Task 3 : Drain mode

### Question 1
> Take a screenshot of the Step 5 and tell us which node is answering.

![](media/T3.1.PNG)

On remarque qu'uniquement `s1` répond aux requêtes.

### Question 2
> Based on your previous answer, set the node in DRAIN mode. Take a screenshot of the HAProxy state page.

Voici les commandes effectués :

![](media/T3.2.PNG)

On remarque alors dans la capture ci-dessous que `s1` est bien en `DRAIN MODE`.

![](media/T3.2-2.PNG)

### Question 3
> Refresh your browser and explain what is happening. Tell us if you stay on the same node or not. If yes, why? If no, why?

![](media/T3.3.PNG)

On va rester sur `s1` tant que l'on recharge la page. Cependant, si l'on ouvre une nouvelle connexion elle sera automatiquement redirigée vers `s2`.

C'est ainsi que se comporte le `drain mode` : il garde toute les connexions déjà existante dessus et les nouvelles seront rédirigées vers un autre serveur.

### Question 4
> Open another browser and open http://192.168.42.42. What is happening?

Nous sommes toujours rédirigés sur `s1` : cela signifie que nous avons toujours le cookie dans le navigateur ayant comme serveur `s1`.

### Question 5
> Clear the cookies on the new browser and repeat these two steps multiple times. What is happening? Are you reaching the node in DRAIN mode?

Non, nous pouvons uniquement atteindre `s2`. Ce comportement est totalement juste car n'ayant plus de cookie relié à `s1` il est désormais pas possible d'atteindre ce dernier (étant toujours en `drain mode`).

![](media/T3.5.PNG)

![](media/T3.5-2.PNG)

### Question 6
> Reset the node in READY mode. Repeat the three previous steps and explain what is happening. Provide a screenshot of HAProxy's stats page.

Voici la commande utilisée :

![](media/T3.6-2.PNG)

![](media/T3.6.PNG)

Après avoir mis en mode `ready` `s1`, on recharge la page. On remarque que l'on reste toujours sur `s2` : et ceci même si l'on ferme et ouvre le navigateur. 

L'explication est que nous avons mis en place les *Sticky Sessions* à la tâche 2 !

Pour changer de serveur nous avons dû effacer les cookies et recharger la page : nous avons ensuite atteint à nouveau `s1`.

![](media/T3.6-3.PNG)

### Question 7
> Finally, set the node in MAINT mode. Redo the three same steps and explain what is happening. Provide a screenshot of HAProxy's stats page.

Voici la commande utilisée :

![](media/T3.7.PNG)

Dans ce mode on sera automatiquement et immediatement rédirigés vers `s2` (et ceci peut importe que notre cookie ayant `s1` comme valeur ou non).

Il n'est alors plus possible d'atteindre `s1` car il est en mode `maint`.

![](media/T3.7-2.PNG)

## Task 4: Round robin in degraded mode

<u>**A NOTER :**</u> Nous avons modifié pour cette tâche différents paramètres du script utilisé sur *JMETER*, on a desormais **4 utilisateurs avec 100 requêtes.**

### Question 1
> Make sure a delay of 0 milliseconds is set on `s1`. Do a run to have a baseline to compare with in the next experiments.

<u>**A NOTER :**</u> *Pour les commandes suivantes, nous nous sommes directement connecté sur la console de `s1` , respectivement `s2`, pour effectuer les commandes suivantes. Nous avons utilisé comme adresse IP `127.0.0.1` sur `s1` et `s2`. Si l'on avait directement les adresses IP (et non pas comme dans notre cas uniquement `localhost`, on aurait pu utilisé comme adresse IP celle de chaque serveur respective : et ceci directement depuis une console de notre machine (sans devoir se connecter au terminal de `s1`, `s2`.*

Les commandes sont les suivantes (on s'assure que le delai soit aussi de *0* sur `s2`) :

````
#Sur s1
curl -H "Content-Type: application/json" -X POST -d '{"delay": 0}' http://127.0.0.1:3000/delay
````
````
#Sur s2
curl -H "Content-Type: application/json" -X POST -d '{"delay": 0}' http://127.0.0.1:3000/delay
````

![](media/T4.1.PNG)

On remarque alors dans la capture ci-dessus que les requêtes sont distribuées de manière équitables entre `s1` et `s2`.

### Question 2

> Set a delay of 250 milliseconds on `s1`. Relaunch a run with the JMeter script and explain what is happening.

Commande sur `s1`:
````
#Sur s1
curl -H "Content-Type: application/json" -X POST -d '{"delay": 250}' http://127.0.0.1:3000/delay
````

![](media/T4.2.PNG)

On remarque alors dans la capture ci-dessus que les requêtes sont distribuées de manière équitables entre `s1` et `s2`.  On va noter par contre que les requêtes sur `s1` sont arrivées bien plus tard/lentement que `s2` (dû au délai sur `s1`). On en déduit qu'il y a une perte de performance, mais aussi que le délai n'est pas assez grand pour pouvoir perturber la distribution des requêtes.

### Question 3

> Set a delay of 2500 milliseconds on `s1`. Same than previous step.

````
#Sur s1
curl -H "Content-Type: application/json" -X POST -d '{"delay": 2500}' http://127.0.0.1:3000/delay
````

![](media/T4.3.PNG)

On remarque alors dans la capture ci-dessus que les requêtes vont uniquement sur `s2`. Dans ce cas le délai est trop important pour qu'une requête puisse atteindre `s1` : il ne sera jamais atteint.

### Question 4

> In the two previous steps, are there any errors? Why?

Non, nous avons eu aucune erreur. Le load balancer redirige bien les requêtes entre les 2 serveurs en fonction du *Round Robin*. Dans ce cas, il va envoyer la première requête à `s1` qui va la traiter et pendant ce temps-là , `s2` va traiter toutes les requêtes (à cause du délai sur `s1`).

### Question 5
> Update the HAProxy configuration to add a weight to your nodes. For that, add `weight [1-256]` where the value of weight is between the two values (inclusive). Set `s1` to 2 and `s2` to 1. Redo a run with a 250ms delay.

Voici les modifications apportés dans le fichier de configuration de *HAProxy :*

![](media/T4.5.PNG)

### Question 6
> Now, what happens when the cookies are cleared between each request and the delay is set to 250ms? We expect just one or two sentence to summarize your observations of the behavior with/without cookies.

<u>**A NOTER :**</u> Nous avons modifié pour cette question différents paramètres du script utilisé sur *JMETER*, on a desormais **10 utilisateurs** et non plus **4**. (afin de pouvoir mieux comparer comme dans un cas réel.)

Voici la capture avec les cookies :

![](media/T4.6.PNG)

Voici la capture sans les cookies :

![](media/T4.6-2.PNG)

Nous constatons une grande différence entre ces deux captures (avec/sans cookies). 

On remarque que lorsqu'on a un serveur lent avec un poids plus grand que le serveur rapide, les cookies sont gardés, les performances vont être ralenties. Tandis que, dans le cas où les cookies ne sont pas gardés, le serveur le plus rapide va pouvoir effectuer bien plus de requêtes (dans ce cas `s2`). 

## Task 5 : Balancing strategies

<u>**A NOTER :**</u> Nous avons enlevé les *poids* sur chaque serveur dans pour cette tâche. Le délai de 250ms reste par contre inchangé. On va alors effectuer les mêmes opérations que la [Task 4, question 6](#Question 6).

### Question 1

> Briefly explain the strategies you have chosen and why you have chosen them.

Nous constatons qu'il existe une multitude d'algorithmes de *Load-balancing*, dont 2 qui on retenu notre attention :

- `leastconn` : Dans cet algorithme, le serveur avec le moins de connexions reçoit la connexion. Le but est alors de pouvoir garantir que tous les serveurs seront utilisés.
- `source`: Dans cet algorithme, *l'adresse IP* source est hachée et divisée par le poids total des serveurs en cours d'exécution pour désigner le serveur qui recevra la demande.

### Question 2
> Provide evidence that you have played with the two strategies (configuration done, screenshots, ...)

Voici la modification effectuée dans le fichier de configuration de *HAProxy* pour tester `leastconn` (*dans `backend node`*):

![](media/T5.2.PNG)

Voici la modification effectuée dans le fichier de configuration de *HAProxy* pour tester `source` (*dans `backend node`*):

![](media/T5.2-2.PNG)

#### Algorithme Leastconn

(Résultats obtenus en supprimant le *delay* de 250ms sur `s1`)

Avec cookies :

![](media/T5.2least_cookie.PNG)

Sans Cookies :

![](media/T5.2least_nocookie.PNG)

Lors de notre exécution de cet algorithme, nous avons remarqué qu'avec les cookies, la charge était bien divisée entre les serveurs. Néanmoins, lors de l'essai sans les cookies, nous avons constaté que très peu de requêtes était effecuté par `s1` (1/10  requête en moyenne). Nous avons alors essayé de supprimer le délai afin de pouvoir comparer uniquement la performance de l'algorithme sans ajouter d'autres variables qui pourrait conditionner nos résultats (comme dans le cas présent).

On va alors constater que le load balancer va bien répartir les charges à part égales (à 3 requêtes près !)

#### Algorithme Source

Avec cookies :

![](media/T5.2source_cookie.PNG)

Sans Cookies :

![](media/T5.2source_nocookie.PNG)

Les deux algorithmes vont produire le même résultat (avec ou sans cookies). On remarque alors qu'uniquement `s2` va être utilisé pour effectuer les requêtes, car l'algorithme de hash à choisi ce dernier.

Cela soulève différents problèmes, comme par exemple dans le cas d'une grande entreprise (même adresse IP publique) qui voudrait accéder à l’application. Tous les utilisateurs serions tous redirigés vers le même serveur et donc le load balancing ne pourrait pas fonctionner comme il le devrait.

### Question 3
>Compare the two strategies and conclude which is the best for this lab (not necessary the best at all).

Selon nous, `leastconn` parait bien plus adapté pour des sessions de longue durée (telles que SQL, LDAP, ...) que de courte (telle que HTTP).

Quant à l'algorithme `source`, il permet une redirection toujours vers le même serveur : ce qui n'est pas forcément une mauvaise chose. Néanmoins, si nos serveurs changent souvent d'adresse *IP*, il devra recalculer souvent les *Hashs* : cela prend du temps et donc ralenti les performances.



## Conclusion

Ce laboratoire permet de voir en pratique ce que nous avons pu voir lors des cours d’AIT (notamment dans le cadre du *Load balancing*).

On en conclu que les *Sticky Sessions* permettent vraiment de pouvoir modifier le comportement du *HAProxy* dans ce cas, de même que pour les différents algorithmes proposés.

Le délai est aussi a prendre en compte, car il permet carrément d'exclure un serveur à partir d'une valeur trop grande !