## RideCleanisingExercise
The task of the "Taxi Ride Cleansing" exercise is to cleanse a stream of TaxiRide events by removing events that start or end outside of New York City.

```python
val rides = env.addSource(rideSourceOrTest(new TaxiRideSource(input, maxDelay, speed)))
val filteredRides = rides.filter(r => GeoUtils.isInNYC(r.startLon, r.startLat) && GeoUtils.isInNYC(r.endLon, r.endLat))
```
Сначала получаем значение поездок, а потом ипользум функции filter и параметра isInNYC, убераем поездки, которые начинаются или заканчиваются в Нью-Йорке.
## RidesAndFaresExercise
The goal of this exercise TaxiRide and TaxiFare is to join together the and records for each ride.
```python
class EnrichmentFunction extends RichCoFlatMapFunction[TaxiRide, TaxiFare, (TaxiRide, TaxiFare)] {
    lazy val rideState: ValueState[TaxiRide] = getRuntimeContext.getState(
      new ValueStateDescriptor[TaxiRide]("saved ride", classOf[TaxiRide]))
    lazy val fareState: ValueState[TaxiFare] = getRuntimeContext.getState(
      new ValueStateDescriptor[TaxiFare]("saved fare", classOf[TaxiFare]))

    override def flatMap1(ride: TaxiRide, out: Collector[(TaxiRide, TaxiFare)]): Unit = {
      val fare = fareState.value
      if (fare != null) {
        fareState.clear()
        out.collect((ride, fare))
      }
      else {
        rideState.update(ride)
      }
    }

    override def flatMap2(fare: TaxiFare, out: Collector[(TaxiRide, TaxiFare)]): Unit = {
      val ride = rideState.value
      if (ride != null) {
        rideState.clear()
        out.collect((ride, fare))
      }
      else {
        fareState.update(fare)
      }
    }
  }
```
В созданном классе мы будем соединять пары <TaxiRide, TaxiFare> по ключу rideId, используя функции flatMap1 и flatMap2 , которые принимают на вход параметры TaxiRide или TaxiFare соответственно. Если в поле класса содержится значение taxiFare или taxiRide, то применяется out: Collector с переданным набором из 2-ух элементов, иначе поданная на вход var записывается в поле класса.
## HourlyTipsExerxise
The task of the "Hourly Tips" exercise is to identify, for each hour, the driver earning the most tips.
```python
    val hourlyTips = fares
      .map((f: TaxiFare) => (f.driverId, f.tip))
      .keyBy(_._1)
      .timeWindow(Time.hours(1))
      .reduce(
        (f1: (Long, Float), f2: (Long, Float)) => { (f1._1, f1._2 + f2._2) },
        new WrapWithWindowInfo())

    val hourlyMax = hourlyTips
      .timeWindowAll(Time.hours(1))
      .maxBy(2)
```
Нахождение водителя с максимальными чаевыми выполняется с помощью данных параметров(fares, hourlyTips). При использовании fares мы выбираем параметры и находим общее число чаевых для водителя. Потом с помощью timeWindowAll и maxBy находим наибольшее значение по 2-ум полям.
## ExpiringStateExercise
The goal for this exercise is to enrich TaxiRides with fare information.
```python
class EnrichmentFunction extends KeyedCoProcessFunction[Long, TaxiRide, TaxiFare, (TaxiRide, TaxiFare)] {
    lazy val rideState: ValueState[TaxiRide] = getRuntimeContext.getState(
      new ValueStateDescriptor[TaxiRide]("saved ride", classOf[TaxiRide]))
    lazy val fareState: ValueState[TaxiFare] = getRuntimeContext.getState(
      new ValueStateDescriptor[TaxiFare]("saved fare", classOf[TaxiFare]))

    override def processElement1(ride: TaxiRide,
                                 context: KeyedCoProcessFunction[Long, TaxiRide, TaxiFare, (TaxiRide, TaxiFare)]#Context,
                                 out: Collector[(TaxiRide, TaxiFare)]): Unit = {
      val fare = fareState.value
      if (fare != null) {
        fareState.clear()
        context.timerService.deleteEventTimeTimer(ride.getEventTime)
        out.collect((ride, fare))
      }
      else {
        rideState.update(ride)
        context.timerService.registerEventTimeTimer(ride.getEventTime)
      }
    }

    override def processElement2(fare: TaxiFare,
                                 context: KeyedCoProcessFunction[Long, TaxiRide, TaxiFare, (TaxiRide, TaxiFare)]#Context,
                                 out: Collector[(TaxiRide, TaxiFare)]): Unit = {
      val ride = rideState.value
      if (ride != null) {
        rideState.clear()
        context.timerService.deleteEventTimeTimer(ride.getEventTime)
        out.collect((ride, fare))
      }
      else {
        fareState.update(fare)
        context.timerService.registerEventTimeTimer(fare.getEventTime)
      }
    }

    override def onTimer(timestamp: Long,
                         ctx: KeyedCoProcessFunction[Long, TaxiRide, TaxiFare, (TaxiRide, TaxiFare)]#OnTimerContext,
                         out: Collector[(TaxiRide, TaxiFare)]): Unit = {
      if (fareState.value != null) {
        ctx.output(unmatchedFares, fareState.value)
        fareState.clear()
      }
      if (rideState.value != null) {
        ctx.output(unmatchedRides, rideState.value)
        rideState.clear()
      }
    }
  }
```
В этом задании надо исправить ошибку 2 задания. Проблема была в том, что данные пары искались бесконечно, пока не заканчивались данные. Поэтому для решения данной задачи будем использовать таймер. При помощи таймера ищем пару значений, только определенного времени, после этого, если мы не нашли пару, записываем значение в объекты с тегами.
