# 1 Задание
1) Создаем класс, описывающий действия каждого философа(внутри находятся данные ID данного философа и ID соседних, информация о соседних вилках и "путь" до вилок).
```python
class Philosopher(Process):
	def __init__(self, task_name: str, id: int, fork_path: str, eat_seconds: int = 15, max_id: int = 5):
		super().__init__()
		self.root = task_name
		self.fork = fork_path
		self.id = id
		self.left_fork_id = id
		self.right_fork_id = id + 1 if id + 1 < max_id else 0
		self.eat_seconds = eat_seconds
		self.partner_id_left = id - 1 if id - 1 >= 0 else max_id-1
		self.partner_id_right = id + 1 if id + 1 < max_id else 0
```
2) Использовать вилки может только один философ, поэтому если кто-то берет вилки, то стол блокируется (общий процесс)
```python
table_lock = zk.Lock(f'{self.root}/table', self.id)
left_fork = zk.Lock(f'{self.root}/{self.fork}/{self.left_fork_id}', self.id)
right_fork = zk.Lock(f'{self.root}/{self.fork}/{self.right_fork_id}', self.id)
```
3) В течении времени eat_seconds будет эмулироваться программа для каждого философа.
```python
start = time()
while time() - start < self.eat_seconds:
```
4) Вилки блокируются если философ поел не больше, чем его соседи и вилки никто не использует.
```python
with table_lock:
	if len(left_fork.contenders()) == 0 and len(right_fork.contenders()) == 0 \
		and counters[self.partner_id_right] >= counters[self.id] \
		and counters[self.partner_id_left] >= counters[self.id]:
			left_fork.acquire()
			right_fork.acquire()
```
5) Если вилки используются, значит философ кушает, когда он поест вилки станут свободными и счетчик еды для него увеличиваем, а если философ не ест, значит он думает.
```python
if left_fork.is_acquired and right_fork.is_acquired:
	print(f'Философ {self.id}: кушает')
	counters[self.id] += 1
	sleep(MEAL_TIME_SEC)
	left_fork.release()
	right_fork.release()
else:
	print(f'Философ {self.id}: думает')
	sleep(WAITING_TIME_SEC)
```
6) Создаем необходимые процессы и задаем константы.
```python
master_zk = KazooClient()
master_zk.start()
if master_zk.exists('/task1'):
	master_zk.delete('/task1', recursive=True)

master_zk.create('/task1')
master_zk.create('/task1/table')
master_zk.create('/task1/forks')
master_zk.create('/task1/forks/1')
master_zk.create('/task1/forks/2')
master_zk.create('/task1/forks/3')
master_zk.create('/task1/forks/4')
master_zk.create('/task1/forks/5')

root = '/task1'
fork_path = 'forks'
seconds_eat = 20
```
7) Создаем лист куда пишем итоговый результат счетчика еды в философе для каждого философа. Создаем философов и запускаем каждого из них.
```python
counters = Manager().list()
p_list = list()
for i in range(0, 5):
	p = Philosopher(root, i, fork_path, seconds_eat)
	counters.append(0)
	p_list.append(p)
for p in p_list: 
  p.start()
```
# 2 Задание
1)	Создадим класс, содержащий путь до своего узла, id и KazooClient.
```python
def __init__(self, root: str, id: int, zk):
	super().__init__()
	self.url = f'{root}/{id}'
	self.root = root
	self.id = id
	self.zk = zk
```
2)	В данном методе обрабатываются действия, такие как commit, rollback или завершение потока.
```python
def watch_myself(data, stat):
    if data != ACTION_DEAD:
	if(stat.version == 1):
	    sleep(1)
	print(f'Client {self.id} triggered {stat.version}')
	if stat.version != 0:
	    print(f'Client {self.id} do {data.decode()}')
	print(f'Client {self.id} run')
    else:
	print(f'Client {self.id} exit')
```
3)	В методе run происходит случайный выбор действия ACTION/ROLLBACK, а также происходит запуск функции watch_myself.
```python
def run(self):      
	self.zk.start()

	value = ACTION_COMMIT if random.random() > 0.5 else ACTION_ROLLBACK
	print(f'Client {self.id} request {value.decode()}')
	self.zk.create(self.url, value, ephemeral=True)
	print(f'Client {self.id} create')
	datawatcher = DataWatch(self.zk, self.url, watch_myself)

	sleep(WAIT_HARD_WORK_SEC)
	self.zk.stop()
	print(f'Client {self.id} stop')
	self.zk.close()
```
4)	Coordinator следит за работой всех потоков и содержит в себе единственное поле timer, который срабатывает по расписанию.
5)	В методе select_action выбирается действие методом голосования и результат выбора отправляется каждому клиенту.
```python
def select_action():
	tr_clients = coordinator.get_children('/task_2/transaction')
	commit_counter = 0
	abort_counter = 0
	for client in tr_clients:
		commit_counter += int(coordinator.get(f'/task_2/transaction/{client}')[0] == ACTION_COMMIT)
		abort_counter +=  int(coordinator.get(f'/task_2/transaction/{client}')[0] == ACTION_ROLLBACK)

	final_action = ACTION_COMMIT if commit_counter == number_of_clients else ACTION_ROLLBACK
	for client in tr_clients:
		coordinator.set(f'/task_2/transaction/{client}', final_action)
````
6)	Метод check_clients срабатывает по таймеру: получает информацию о клиентах и оповещает других, если один из уже подключенных клиентов отсоединился и рассылает им предупреждение об этом.
```python
def check_clients():
	tr_clients = coordinator.get_children('/task_2/transaction')
	for i in range(len(Coordinator.session_logs)):
		if Coordinator.session_logs[i] is True and str(i) not in tr_clients:
			print("Switching off")
			Coordinator.timer.cancel()
			for client in tr_clients:
				coordinator.set(f'/task_2/transaction/{client}', ACTION_DEAD)
			sleep(0.5)
			for client in tr_clients:
				zks[int(client)].stop()
				zks[int(client)].close()
				processes[int(client)].kill()
			sys.exit()
```
7)	Метод watch_clients устанавливает значения в session_logs при первом подключение клиента, а далее происходит обработка количества клиентов.
```python
@coordinator.ChildrenWatch('/task_2/transaction')
	def watch_clients(clients):
		for client in clients:
			Coordinator.session_logs[int(client)] = True

		if len(clients) == 0:
			if Coordinator.timer is not None:
				Coordinator.timer.cancel()
		else:
			if Coordinator.timer is not None:
				Coordinator.timer.cancel()
			Coordinator.timer = threading.Timer(duration, check_clients)
			Coordinator.timer.daemon = True
			Coordinator.timer.start()

		if len(clients) < number_of_clients:
			print(f'Waiting for the others. clients={clients}')
		elif len(clients) == number_of_clients:
			select_action()
```
8)	Создаем общий процесс и клиенты.
```python
Coordinator.session_logs = [False] * number_of_clients
coordinator = KazooClient()
coordinator.start()

if coordinator.exists('/task_2'):
	coordinator.delete('/task_2', recursive=True)

coordinator.create('/task_2')
coordinator.create('/task_2/transaction')

Coordinator.timer = None
root = '/task_2/transaction'
        
        for i in range(number_of_clients):
            zks.append(KazooClient())
            process = Client(root, i, zks[-1])
            processes.append(process)
            process.start()
            sleep(5)
```
