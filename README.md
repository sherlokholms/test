<?php
/*Ниже представлен класс для удобного ведения логов. Логи содержат дату и время события (в формате YYYY-MM-DD HH:MM:SS), 
сообщение логирования (строка, массив, объект, исключение). Логирование осуществляется с использованием принципа 
полиморфизма - создан абстрактный класс Logger, в котором объявлен абстрактный метод abstract protected function write_to_log($message);
*/

require_once 'config.php';//Подключение файла с настройками соединения с БД 

abstract class Logger
{  
    abstract protected function write_to_log($message);// Данный метод должен быть определён в дочернем классе 
}
//---------------------------Запись в текстовый файл--------------------------//
class File extends Logger
{
    public function write_to_log($message) 
	{
			$name='log';
			if(strlen(trim($message))<2)
			{
				return false;
			}
			if(preg_match("/^([_a-z0-9A-Z]+)$/i",$name, $matches))//проверка имени файла (имя файла должно состоять только из латинских букв, цифр и знака подчеркивания)
			{
				$file_path = $_SERVER['DOCUMENT_ROOT'].'/logs/'.$name.'.txt';//местоположение лог-файла
				$text = '['.date('Y-m-d H:i:s').']'." - ".htmlspecialchars($message)."\r\n";//Добавление даты и времени
				$handle = fopen($file_path, "a+");// автоматически создается файл, если он не найден.
				@flock($handle,LOCK_EX);
				fwrite($handle,$text);
				fwrite($handle,
				"================================================================\r\n\r\n");
				@flock($handle,LOCK_UN);
				fclose($handle);
			}
	}	
}
//---------------------------Запись в базу данных----------------------------//	
class Database extends Logger
{
	public function write_to_log($message)
		{	
			$date = date('Y-m-d H:i:s');
			$this->bdname="log_database";
			$this->tbname="log_table";
			(!mysql_pconnect("localhost", "root", "")) exit(mysql_error());
            $r = mysql_query("CREATE DATABASE IF NOT EXISTS $this->bdname");// автоматически создается база данных, если она не найдена.
            if (!$r) exit(mysql_error());
            mysql_select_db($this->bdname);
            mysql_query('SET NAMES UTF8');
            $res = mysql_query("CREATE TABLE IF NOT EXISTS $this->tbname
			(`datetime` DATETIME COLLATE utf8_general_ci NOT NULL,
		     `message` CHAR(200) COLLATE utf8_general_ci NOT NULL);");
			$sql	="INSERT INTO 
					$this->tbname
					VALUES(
						'$date',
						'$message'
						)";
			mysql_query($sql) or die(say_error("Error: ",  mysql_error()));
		}	
}
//-----------------------------Запись в stdout-------------------------------//	
class Stdout extends Logger
{
	public function write_to_log($message)
		{
			$text = '['.date('Y-m-d H:i:s').']'." - ".htmlspecialchars($message)."\r\n";//Добавление даты и времени
			$fp = fopen("php://stdout", 'r+');
            fputs($fp, $text);
			
            // Читаем то, что мы записали. 
            rewind($fp);
            echo stream_get_contents($fp);		
		}		
}
//------------------------Использование класса-------------------------------//
$class1 = new File;
$class2 = new Database;
$class3 = new Stdout;
$class1->write_to_log('new_message');echo "Лог был записан в текстовый файл log.txt";
//$class2->write_to_log('new_message'); echo "Лог был записан в базу данных";	
//$class3->write_to_log('new_message'); echo "Лог был записан в стандартный поток вывода stdout";		
?>
