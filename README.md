Задачи, которые выполняются в пуле (родительские задачи), могут пробросить в потоки собственные подзадачи. Например, вспомните параллельную быструю сортировку из 28 модуля. В этом алгоритме две сортируемые части массива независимы и могут быть обработаны параллельно.

В 28 модуле для левой части разбитого массива мы запускали рекурсию в потоке. Теперь нужно сделать так, чтобы левая часть алгоритма пробрасывалась в тот же пул потоков, в котором в данный момент выполняется задание на сортировку отдельной части массива.

Возникает проблема, которую придется решить: мы не можем больше ожидать завершения выполнения сортировки левой части массива. Ниже приведен код из модуля 28: 
------
   if(make_thread && (right_bound - left > 10000))
   {
       // если элементов в левой части больше чем 10000
       // вызываем асинхронно рекурсию для правой части
       auto f = async(launch::async, [&]() {
           quicksort(array,left,right_bound);
       });
       quicksort(array, left_bound, right);
   } else {
       // запускаем обе части синхронно
       quicksort(array, left, right_bound);
       quicksort(array, left_bound, right);
   }
-------
Здесь при вызове деструктора ~f поток, который вызвал async, блокируется, пока не завершится созданный им поток. Перепишем код, учитывая, что теперь у нас есть пул потоков:
-------
   if(make_thread && (right_bound - left > 10000))
   {
       auto f = pool.push_task(quicksort,array,left,right_bound);
       quicksort(array, left_bound, right);
       f.wait(); // нельзя так написать
   } else {
       // запускаем обе части синхронно
       quicksort(array, left, right_bound);
       quicksort(array, left_bound, right);
   }
-------
Если мы напишем так, то поймаем deadlock или мертвую блокировку. Ведь получится, что каждый из потоков в пуле вызовет f.wait(), и ни один поток в итоге не будет выполняться, ожидая завершения задания другого, в то время как другой поток так же ждёт. Например, при всего двух потоках в пуле первый поток забросит подзадачу в пул, её подхватит второй поток, который простаивает, затем так же забросит подзадачу в пул. Но эту подзадачу некому обрабатывать, потоки зависли в ожидании. Получаем заклинивший пул.

Но нам нужно каким-то образом отследить момент окончания выполнения параллельной сортировки. Вам предстоит решить эту проблему самостоятельно.

Нужно придумать механизм, который переводил бы родительскую задачу в состояние готовности (фьючерс) тогда, когда все ее подзадачи были совершены.

Задачу можно решить, используя std::promise и std::shared_ptr, где последний поможет в качестве счетчика ссылок на родительскую задачу. За основу возьмите пул потоков из третьего юнита с концепцией work stealing.

Продемонстрируйте скорость работы алгоритма быстрой сортировки при использовании пула потоков. Учитывайте, что не стоит запускать задачу на вычисление левой части алгоритма в качестве подзадачи, если количество элементов в этой части не превосходит 100000 (расходы на захват мьютекса очень велики).
