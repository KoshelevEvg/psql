Триггер для записи кол-во занятых мест на каждом автобусе 

```sql
CREATE OR REPLACE FUNCTION update_bus_occupancy()
RETURNS TRIGGER AS $$
BEGIN
    -- Обновляем поле spec в таблице bus, увеличивая счетчик занятых мест
    UPDATE bus 
    SET spec = jsonb_set(
        COALESCE(spec, '{}'::jsonb),
        '{occupied_seats}',
        COALESCE((spec->>'occupied_seats')::int, 0)::text::jsonb
    WHERE id = (
        SELECT r.fkbus 
        FROM ride r 
        WHERE r.id = NEW.fkride
    );
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_bus_occupancy
AFTER INSERT ON tickets
FOR EACH ROW
EXECUTE FUNCTION update_bus_occupancy();
```

Получается, что при добавлении новой записи в таблицу билетов, будет добавлять значение занятых мест в каждом из автобусе.

### Оценка производительности
Без триггера:
- Простая вставка в таблицу `tickets` - это одна операция `INSERT`
- Время выполнения зависит только от размера таблицы и индексов

С триггером
- Вставка в таблицу `tickets` (как и раньше)
- `SELECT` для получения `fkbus` из таблицы `ride`
- `UPDATE` таблицы `bus` с `jsonb` операцией
- Все это для каждой вставляемой строки

Так же прикладываю триггеры для добавление мест на случай переполнения автобуса и триггер для обновления данных при удалении билетов
```sql
CREATE OR REPLACE FUNCTION update_bus_occupancy_current()
    RETURNS TRIGGER AS
$$
DECLARE
    bus_record     jsonb;
    current_seats  TEXT;
    total_seats    INT;
    occupied_seats INT;
    initial_count  INT;
    bus_id         INT;
BEGIN
    -- Получаем ID автобуса и его текущий spec

    SELECT id, spec INTO bus_id, bus_record
    FROM bus
    WHERE id = (
        SELECT r.fkbus FROM ride r WHERE r.id = NEW.fkride
    );


    RAISE NOTICE 'Триггер сработал для fkride: %', bus_id;

    -- Проверяем, что запись найдена
    IF bus_id IS NULL THEN
        RAISE EXCEPTION 'Bus not found for ride ID %', NEW.fkride;
    END IF;

    -- Проверяем наличие поля Bus
    current_seats := bus_record ->> 'Bus';

    RAISE NOTICE 'fmt current_seats %', current_seats;
    IF current_seats IS NULL THEN
        RAISE EXCEPTION 'Field "Bus" not found in bus specification';
    END IF;

    -- Парсим общее количество мест
    total_seats := SPLIT_PART(current_seats, '-', 1)::INT;

    -- Получаем текущее количество проданных билетов для этого рейса
    SELECT COUNT(*) INTO initial_count
    FROM tickets
    WHERE fkride = NEW.fkride;

    -- Инициализируем occupied_seats
    IF bus_record ? 'occupied_seats' THEN
        -- Если поле есть, берём его значение
        occupied_seats := (bus_record->>'occupied_seats')::INT;

        -- Синхронизируем с реальным количеством билетов (на случай расхождений)
        IF occupied_seats < initial_count THEN
            occupied_seats := initial_count;
        END IF;
    ELSE
        -- Если поля нет, используем количество билетов как начальное значение
        occupied_seats := initial_count;

        -- Добавляем поле в JSON
        bus_record := jsonb_set(bus_record, '{occupied_seats}', to_jsonb(occupied_seats));
    END IF;

    -- Увеличиваем счетчик занятых мест
    occupied_seats := occupied_seats + 1;

    -- Проверяем доступность мест
    IF occupied_seats > total_seats THEN
        RAISE EXCEPTION 'No available seats on this bus (Total seats: %, Occupied: %)',
            total_seats, occupied_seats;
    END IF;

    -- Обновляем запись в bus
    UPDATE bus
    SET spec = jsonb_set(
            bus_record,
            '{occupied_seats}',
            to_jsonb(occupied_seats))
    WHERE id = bus_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER trg_update_bus_occupancy_current
    BEFORE INSERT
    ON tickets
    FOR EACH ROW
EXECUTE FUNCTION update_bus_occupancy_current();

CREATE OR REPLACE FUNCTION update_bus_occupancy_current_on_delete()
    RETURNS TRIGGER AS
$$
DECLARE
    bus_record     jsonb;
    current_seats  TEXT;
    total_seats    INT;
    occupied_seats INT;
    initial_count  INT;
    bus_id         INT;
BEGIN
    -- получаем ID пассажира и его bus_ID
    SELECT fkride INTO bus_id
    FROM tickets
    WHERE id = OLD.id;

    -- Получаем ID автобуса и его текущий spec
    SELECT spec INTO bus_record
    FROM bus
    WHERE id = bus_id;

    -- Проверяем, что запись найдена
    IF bus_id IS NULL THEN
        RAISE EXCEPTION 'Bus not found for ride ID %', NEW.fkride;
    END IF;

    -- Проверяем наличие поля Bus
    current_seats := bus_record ->> 'Bus';

    RAISE NOTICE 'fmt current_seats %', current_seats;
    IF current_seats IS NULL THEN
        RAISE EXCEPTION 'Field "Bus" not found in bus specification';
    END IF;

    -- Парсим общее количество мест
    total_seats := SPLIT_PART(current_seats, '-', 1)::INT;

    -- Получаем текущее количество проданных билетов для этого рейса
    SELECT COUNT(*) INTO initial_count
    FROM tickets
    WHERE fkride = NEW.fkride;

    -- Инициализируем occupied_seats
    IF bus_record ? 'occupied_seats' THEN
        -- Если поле есть, берём его значение
        occupied_seats := (bus_record->>'occupied_seats')::INT;

        -- Синхронизируем с реальным количеством билетов (на случай расхождений)
        IF occupied_seats < initial_count THEN
            occupied_seats := initial_count;
        END IF;
    ELSE
        -- Если поля нет, используем количество билетов как начальное значение
        occupied_seats := initial_count;

        -- Добавляем поле в JSON
        bus_record := jsonb_set(bus_record, '{occupied_seats}', to_jsonb(occupied_seats));
    END IF;

    -- Уменьшаем счетчик занятых мест
    occupied_seats := GREATEST(0, occupied_seats - 1);;

    -- Проверяем доступность мест
    IF occupied_seats > total_seats THEN
        RAISE EXCEPTION 'No available seats on this bus (Total seats: %, Occupied: %)',
            total_seats, occupied_seats;
    END IF;

    -- Обновляем запись в bus
    UPDATE bus
    SET spec = jsonb_set(
            bus_record,
            '{occupied_seats}',
            to_jsonb(occupied_seats))
    WHERE id = bus_id;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER trg_update_bus_occupancy_current_delete
    BEFORE DELETE
    ON tickets
    FOR EACH ROW
EXECUTE FUNCTION update_bus_occupancy_current_on_delete();
```