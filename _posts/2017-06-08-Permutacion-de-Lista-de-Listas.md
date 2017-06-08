---
layout: post
title: Permutacion de Lista de Listas
subtitle: Permutacion de Lista de Listas
---

Hola, después de bastante tiempo vengo con un post corto, es un algoritmo que me toco realizar como "prueba" para la postulación a un trabajo.<br>
Resulta que debía calcular la permutacion de una lista de listas, ya habia hecho algo similar, pero solo para una lista plana, no una lista que contenta listas.
<br>
Finalmente como desafío, y sin buscar códigos de ejemplo me decidí a realizarlo para C#, después de intentarlo, y lograrlo con un código que de elgante tenía muy poco, me decidí a buscar en google, con pocos resultados, y los que habían eran para otros lenguajes. <br>
Este es el resultado, para quien le interese, esta en C# pero es facilmente portable.

la entrada es 
```cs
[[1,3],[‘a’],[4,5]] 
```
y la salida
```cs
[[1,a,4] [1,a,5] [3,a,4] [3,a,5]]
```

```cs
        public static List<List<object>> GetPermutation(List<List<object>> input)
        {
            const int skipFrom = 1;

            List<List<object>> result = new List<List<object>>();
            if (!input.Any())
            {
                result.Add(new List<object>());
                return result;
            }

            List<object> firsList = input.First();

            int pendingLists = input.Count - 1;

            List<List<object>> restLists = GetPermutation(input.GetRange(skipFrom, pendingLists));//recursion

            //recorre la primera lista
            firsList.ForEach(fl =>
            {
                //recorre el resto de elementos
                restLists.ForEach(re =>
                {
                    List<object> resultElement = new List<object>();
                    //añade el item del primer elemento
                    resultElement.Add(fl);
                    re.ForEach(item => resultElement.Add(item));
                    result.Add(resultElement);
                });
            });

            return result;
        }
```

Espero les sirva,   

Saludos!


Saludos!<br>
John.