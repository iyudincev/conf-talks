# ApolloClient 3 <!-- .element: class="grey" -->

# Garbage collection

-----

## Он наконец-таки появился 🎉

-----

## Как работает GC? <!-- .element: class="green" -->

- Начиная с ROOT_QUERY, бежит вглубь, собирая __ref в какой-нить Set <!-- .element: class="fragment" -->
- Вторым проходом пробегается по всем ключам в нормализованном кэше и удаляет те записи: <!-- .element: class="fragment" -->
  - которых нет в Set'е <!-- .element: class="fragment" -->
  - и нет в retain-set'е <!-- .element: class="fragment" -->

-----

### Ручная чистка кэша <!-- .element: class="green" -->
  
<div class="fragment"><code>cache.retain(id)</code> – защитит запись от удаления из GC</div>
<div class="fragment"><code>cache.release(id)</code> – снимет отметку защиты от удаления</div>
<div class="fragment"><code>cache.evict(id)</code> – сразу удалит запись из кэша (даже если запись защищена в <code>retain-set'e</code>)</div>

-----

### Автоматом GC `будет` вызываться, когда компоненты от-монтируются.

На хуках они используют useEffect, <br />чтоб поймать факт де-монтировки. <!-- .element: class="fragment orange" -->
