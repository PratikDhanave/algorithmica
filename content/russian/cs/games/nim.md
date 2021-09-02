---
title: Игра «Ним»
weight: 5
---

## Ним

Два игрока играют в следующую игру - есть $n$ кучек камней, в каждой -
$a_{i}$ камней. За один ход любой игрок может достать из любой кучки $c
\> 0$ камней, выигрывает игрок, сделавший последний ход.

Данная игра называется ним и ниже мы докажем, что любая равноправная
цикличная игра сводится к ниму.

## Решение нима

Сведем ним из нескольких кучек к ниму из одной. Первый игрок выигрывает
тогда и только тогда, когда $a_{1} \\bigoplus \\dots \\bigoplus a_{n}
\\neq 0$($a_{1} \\bigoplus \\dots \\bigoplus a_{n}$ - называют
$XOR$-суммой).

Докажем по индукции по суммарному количеству камней в кучках:

1\) Если у нас есть только кучки размера 0, то их $\\bigoplus$ = 0 и
следовательно положение - проигрышное.

2\) Пусть у нас есть $n$ кучек размера $a_{1} \\dots a_{n}$, покажем,
что из выигрышного положения можно перейти в проигрышное положение и
что из проигрышного положения есть переходы только в выигрышные.

Если $a_{1} \\bigoplus \\dots \\bigoplus a_{n} = 0$ и мы поменяли
размер $i$ кучки на $y$, то теперь наша $XOR$-сумма = $a_{1}
\\bigoplus \\dots y \\dots \\bigoplus a_{n}$, покажем, что $a_{1}
\\bigoplus \\dots y \\dots \\bigoplus a_{n} \\neq 0$. Если $a_{1}
\\bigoplus \\dots y \\dots \\bigoplus a_{n} = 0$, то $a_{1} \\bigoplus
\\dots y \\dots \\bigoplus a_{n} \\bigoplus a_{1} \\bigoplus \\dots
\\bigoplus a_{n} = 0 \\bigoplus 0 = 0$ ($XOR$-сумма до хода $\\bigoplus
XOR$-сумма после хода), уберем из этого выражения все равные элементы и
получим $a_{i} \\bigoplus y = 0$, но так как размер кучки обязательно
изменился и $a_{i} \\neq y$, то противоречие и следовательно из
проигрышной позиции есть переходы только в выигрышные позиции.

Если $a_{1} \\bigoplus \\dots \\bigoplus a_{n} = x \\neq 0$, то
возьмем старший единичный бит $x$, найдем $a_{i}$, у которого
данный бит равен 1 и сделаем ход $a_{i} = a_{i} \\bigoplus x$, так
как $a_{i}$ уменьшилось и $x \\bigoplus x$ = 0, то такой ход всегда
можно сделать и он всегда ведет в проигрышное состояние.

## Ним с прибавлением

Пусть теперь нам в ниме разрешено сделать операцию $a_{i} += x$, при
этом нам гарантируется, что игра - ациклична(например нам разрешено
сделать операцию + не более $n$ раз), тогда такой ним эквивалентен
обычному ниму, так как на ход соперника +x мы можем просто ответить
-x.

## Сведение любой игры к ниму

Пусть есть Ациклический ориентированный граф и игра на нем вида можно
перейти по ребру, последний сделавший ход - выиграл, сведем ее к ниму
:

Пусть $f(u)$ - функция, сопоставляющая вершине $u$ ним размера $f(u)$

Сопоставим листьям ним размера 0, так как и ним размера 0 и листья -
проигрышные позиции.

Для каждого не листа рассмотрим список его сыновей, пусть в этом списке
есть все числа от 0 до $x - 1$(возможно не один раз) и что-то еще(числа
$\> x$), но при этом нет $x$, тогда это значит, что мы стоим в позиции
из которой можно перейти в ним размера от 0 до $x - 1$, а это значит,
что мы стоим в ниме размера $x$. Данная операция(нахождение
минимального неотрицательного числа, которого нет в массиве)
называется mex(minimal excluding).

И в самом деле такое сведение верное, так как из проигрышной вершины(в
которой стоит не 0) нет перехода в проигрышную вершину(иначе mex был
бы не 0). Из выигрышной вершины всегда же есть переход в
проигрышную(так как в сыновьях есть 0).

  - [Примеры задач](Примеры_задач "wikilink")

[Категория:Конспект](Категория:Конспект "wikilink")