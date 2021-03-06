Переход на Luminous
-------------------

#. Делаем всё по инструкции: http://docs.ceph.com/docs/master/releases/luminous/#upgrade-from-jewel-or-kraken

#. Укажите какие пулы для чего будут использоваться
   (``rbd``, ``cephfs`` или ``cephfs_metadata``. Про остальные ничего не знаю.)
   ``ceph osd pool application enable POOLNAME POOLTYPE``

#. Надо как-то там подправить пермишшены (osd blacklist). Там ошибка в документации
   -- слетают пермишшены. Ещё кому-то там надо дать больше прав (писать в мгр?)
   без этого ceph osd df перестает работать. После смены прав что перезапускать?
   Обнаруживается через передеплой MGR/Monitor/OSD. ceph-deploy выставляет другие
   права -- не как было при Кракене.

#. Проблемы с удалением старых снапшотов RBD (known bug). Лечится удалением
   снапшота клиентом от джевел или кракен. TODO: пруф и копия в блоке про RBD.

#. По-моему нужно уcтановить классы OSD. Но они вроде при перезапуске сами
   себя проставят. TODO: команда.

#. Включаем дашборд

   ``ceph mgr module enable dashboard``.
   Возможно, нужно добавить ещё и во тэто  в ceph.conf:

   .. code::

      [mgr]
      mgr_modules = dashboard

   А потом ещё и ``ceph config-key put mgr/dashboard/server_addr ::``. Без этого
   дашборд не заработает.

   Смотрим по ``ceph -s`` какой менеджер активен и подключаемся туда на порт ???? (вписать).


#. Меняем straw -> straw2:

   .. code-block:: sh

      # Создадим полный crush-map и сохраним его во временный файл
      ceph osd getcrushmap | crushtool -d - | sed -r 's/alg straw$/alg straw2/' | crushtool -c /dev/stdin -o newcrush.dat
      # TODO: Перед установкой посмотреть сколько данных будет не на сових местах.
      # Установим его в качестве нового крушмапа.
      ceph osd setcrushmap -i newcrush.dat


#. Оптимизируем CRUSH-map:

   В новых версиях меняются алгоритмы консистентного хеша. Как итог -- меньше
   ребаланса при добавлении/удалении OSD, например, или более равномерное
   распределение по OSD.

   .. warning::

      Это требует повышения минимальной версии до Jewel. Более старые клиенты
      не смогут подключаться к такому кластеры потому что не могут в такое
      хеширование. Возможны промежуточные варианты (чуть получше хеширование,
      но не самое лучшее) -- см. ссылку выше.

   .. warning::

      Не смотря на заявление документации о том что будет перемещение не более
      чем 10% данных, в моём кластере было около 50% данных не на своих местах.

   .. code-block:: sh

      ceph osd set-require-min-compat-client jewel
      ceph osd crush tunables optimal

#. See also

   http://crush.readthedocs.io/en/latest/ceph/optimize.html
