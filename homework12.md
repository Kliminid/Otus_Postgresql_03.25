Cледовал инструкции из развернутого описания ДЗ а затем создал триггер
Триггер срабатывает после операций INSERT, UPDATE или DELETE в таблице sales, также использует информацию из таблицы goods для получения названия товара по его ID.
``` 
postgres=# CREATE OR REPLACE FUNCTION update_goods_sum_mart()
RETURNS TRIGGER AS $$
BEGIN

    IF TG_OP = 'INSERT' THEN

        IF EXISTS (SELECT 1 FROM goods_sum_mart WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id)) THEN

            UPDATE goods_sum_mart
            SET sum_sale = sum_sale + NEW.sale_amount
            WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);
        ELSE

            INSERT INTO goods_sum_mart (good_name, sum_sale)
            SELECT good_name, NEW.sale_amount
            FROM goods
            WHERE goods_id = NEW.good_id;
        END IF;


    ELSIF TG_OP = 'UPDATE' THEN

        IF OLD.good_id <> NEW.good_id THEN

            UPDATE goods_sum_mart
            SET sum_sale = sum_sale - OLD.sale_amount
            WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);


            IF EXISTS (SELECT 1 FROM goods_sum_mart WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id)) THEN
                UPDATE goods_sum_mart
                SET sum_sale = sum_sale + NEW.sale_amount
                WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);
            ELSE
                INSERT INTO goods_sum_mart (good_name, sum_sale)
                SELECT good_name, NEW.sale_amount
                FROM goods
                WHERE goods_id = NEW.good_id;
            END IF;
        ELSE

            UPDATE goods_sum_mart
            SET sum_sale = sum_sale + (NEW.sale_amount - OLD.sale_amount)
            WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);
        END IF;


    ELSIF TG_OP = 'DELETE' THEN

        UPDATE goods_sum_mart
        SET sum_sale = sum_sale - OLD.sale_amount
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);


        DELETE FROM goods_sum_mart
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id)
        AND sum_sale = 0;
    END IF;

    RETURN NULL;
END;
$$ LANGUAGE plpgsql;


CREATE TRIGGER sales_mart_trigger
AFTER INSERT OR UPDATE OR DELETE ON sales
FOR EACH ROW EXECUTE FUNCTION update_goods_sum_mart();
CREATE FUNCTION
CREATE TRIGGER
postgres=#
```