# phpExamples

## 查看、杀Mysql
~~~
  
    function showMysqlLists($request, $response)
    {

        $model = new \model\Member();
        $allLists = $model->query("SHOW FULL PROCESSLIST", true);

        foreach ($allLists as $row) {
            echo "<pre>";
            print_r([
                '语句' => $row['Info'], 
                '当前状态' => $row['Command'] . ' ' . $row['State'],
                '用时' => $row['Time'], 
                'id' => $row['Id'], ]);
            echo "<pre>";

            \yoka\Debug::log('$row', $row);
        }

        if ($request->kill) {
            $model = new \model\Company();
            $rows = $model->query("SHOW processlist", true);
            foreach ($rows as $row) {
                if($request->killSleep){
                    if ( $row['Command'] == "Sleep" ) {
                        \yoka\Debug::log('sleep了', 
                        [
                            'Command' => $row['Command'], 
                            'State' => $row['State'],
                            'Time' => $row['Time'], 
                            'Info' => $row['Info'], 
                            'run' => "KILL {$row['Id']}", 
                        ]);
                        $res = $model->query("KILL {$row['Id']}", true);
                        \yoka\Debug::log('res', $res);
                    }
                }

                else{
                    if ($row['Command'] == "Query" || $row['Command'] == "Sleep" || $row['Command'] ==
                    "Execute") {
                    if (strpos($row['State'], 'lock') !== false) {
                        \yoka\Debug::log('锁住了', [
                            'Command' => $row['Command'], 
                            'State' => $row['State'],
                            'Time' => $row['Time'], 
                            'Info' => $row['Info'], 
                            'run' => "KILL {$row['Id']}", ]);
                        $res = $model->query("KILL {$row['Id']}", true);
                        \yoka\Debug::log('res', $res);
                    } else {
                        if ($request->force) {
                            \yoka\Debug::log('尚未锁住 强行杀死', [
                                'Command' => $row['Command'], 
                                'State' => $row['State'],
                                'Info' => $row['Info'], 
                                'Time' => $row['Time'], 
                                'run' => "KILL {$row['Id']}", ]);
                            $res = $model->query("KILL {$row['Id']}", true);
                            \yoka\Debug::log('res', $res);
                        } else {
                            \yoka\Debug::log('尚未锁住 暂时不杀', [
                                'Command' => $row['Command'], 
                                'State' => $row['State'],
                                'Time' => $row['Time'], 
                                'Info' => $row['Info'], 
                                'run' => "KILL {$row['Id']}", ]);
                        }

                    }

                } else {
                    \yoka\Debug::log($row['Command'] . '', $row);
                }
                }
                

            }
        }

    }


     function killMysqlByKeyword($request, $response)
    {

        $model = new \model\Member();
        $allLists = $model->query("SHOW FULL PROCESSLIST", true);
        if(!trim($request->keyword)){
            return ;
        }
        
        $model = new \model\Company();
        $rows = $model->query("SHOW processlist", true);
        foreach ($rows as $row) {
        
            if(strpos($row['Info'],trim($request->keyword)) !== false){ 
                \yoka\Debug::log('锁住了', ['Command' => $row['Command'], 'State' => $row['State'],
                    'Time' => $row['Time'], 'Info' => $row['Info'], 'run' => "KILL {$row['Id']}", ]);
                $res = $model->query("KILL {$row['Id']}", true);
                \yoka\Debug::log('res', $res);
            }else{
                
            } 

        }

    }
    

    
 SHOW FULL PROCESSLIST  ;
 Select concat('KILL ',id,';') from information_schema.processlist where user='root';

select 
	GROUP_CONCAT(stat SEPARATOR ' ') 
from 
	(
		select 
			concat('KILL ', id, ';') as stat 
		from 
			information_schema.processlist
        WHERE USER = 'root'
	) as stats ;


每秒执行查看进程数
mysqladmin -u root -p  -i 1 processlist 

监控进程数 过多时自动杀死
sudo vi mysql_monitor.php ;
<?php
$mysqli = new mysqli("localhost","root","root","mytest");

// Check connection
if ($mysqli -> connect_errno) {
  echo "Failed to connect to MySQL: " . $mysqli -> connect_error;
  exit();
}
  
$sql = "SELECT 
	* 
	FROM 
	information_schema.processlist 
	WHERE 
	USER = 'root' 
";

if ($result = $mysqli->query($sql)) {
	// 40个以内的连接数不予处理
	if($result -> num_rows<=40){
		// echo "40个以内的连接数不予处理: " . $result -> num_rows;
		exit();
	} 

	// 干掉十秒以上的
	$sql = "SELECT 
			* 
			FROM 
			information_schema.processlist 
			WHERE 
			USER = 'root' 
			AND TIME>=10
	 ";
	
	$result = $mysqli->query($sql);  
	$ids = []; 
	while ( $rows = $result->fetch_assoc() ) {
		print_r(array_merge($rows,[
			'DATE'=>date('Y-m-d H:i:s')
		])); 
		$ids[$rows['ID']] = $rows['ID'];
	} 

	foreach($ids as $id){
		$sql = " KILL  ".$id; 
		$result = $mysqli->query($id); 
	}  
} 
$mysqli -> close();

sudo vi kill.txt ;
sudo chmod 777 kill.txt ;

watch -n 1 "/usr/bin/php mysql_monitor.php >> kill.txt"

~~~

## 本地常驻
~~~
function get()
{


	$js = file_get_contents("http://admin2018.aibangmang.org/?_c=crontab&_a=pullAllCompanysListsNewVersion&limit=1000&province=%E6%B5%99%E6%B1%9F%E7%9C%81&debug=0&");

	return $js;
}
 
ini_set('memory_limit','500M');    // 临时设置最大内存占用为3G
set_time_limit(0);   // 设置脚本最大执行时间 为0 永不过期

while (1) {
	$t1 = microtime(true);
	$n = get();
	sleep(1);
	if (!$n) {
		// sleep(1);
	} 
	echo $n, ' spend ', microtime(true) - $t1, "\n";

}
~~~  

## reflush by ids

~~~

 function getMinId(){
        $model = new self();
        $resModel =$model->fetchOne("SELECT id FROM `agreements__plan` ");
        return  $resModel?$resModel->id:0;
 }

 function getDailyRunedTimes($actionName){
        $cache = \yoka\Cache::getInstance('aibangmang');
        $cacheKey = $actionName.'_runed_time_' . date('Ymd');
        $times = $cache->get($cacheKey);
        return $times?:1;   
}

function setDailyRunedTimes($actionName,$page,$maxPage){
        $cache = \yoka\Cache::getInstance('aibangmang');
        $cacheKey = $actionName.'_runed_time_' . date('Ymd');
        if($page>$maxPage){
            $page =1 ;
        }
        $times = $cache->set($cacheKey,$page);
        return $times?:1;   
}

function reflushIfCanExtend($request, $response){
        $beginTime = microtime(true);

        $limit = $request->limit?:500;
        $page =  self::getDailyRunedTimes('reflushIfCanExtend');
        if($request->page>0){
            $page =  $request->page ;
        }
         
        $minId = \model\AgreementsPlan::getMinId();
        $minId = ($page-1)*$limit+$minId;

         // 业务逻辑

        self::setDailyRunedTimes('reflushIfCanExtend',$page+1,25);

        $endTime = microtime(true);
        $this->crontabSuccess(['msg'=>'更新完成', 'time'=>$endTime-$beginTime]);
}


~~~

## replaceData

~~~ 
static function  replaceData($Data){
        $mode = new self();

        $fieldsStr = \model\agreement\Order::transferColomnArrToStr(array_keys($Data));
        $valuesStr = \model\agreement\Order::transferDbArrToStr($Data);
         
        return  $mode->query("
            REPLACE INTO ".self::$table." (
                {$fieldsStr}
            )
            VALUES {$valuesStr}
           ");
}


static function  replaceDatas($Datas){
        $mode = new self();
        $DatasBak = array_values($Datas);
        $Data = $DatasBak[0];
        $fieldsStr = \model\agreement\Order::transferColomnArrToStr(array_keys($Data));
        $valuesStr = \model\agreement\Order::transferDbArrsToStr($Datas);
         
        return  $mode->query("
            REPLACE INTO ".self::$table." (
                {$fieldsStr}
            )
            VALUES {$valuesStr}
           ");
} 

 static function transferColomnArrToStr($colomnsArr){
        $fieldsStr = "";
        $i = 1;
        $totalColomnsNums = count($colomnsArr);
        foreach($colomnsArr as $field){
            $fieldsStr .= "   `{$field}`  ";
            if($i<$totalColomnsNums){
                $fieldsStr .= "   ,  ";
            }

            $i++;
        }
        return $fieldsStr;
}

static function transferDbArrToStr($colomnsArr){
        $fieldsStr = "";
        $i = 1;
        $totalColomnsNums = count($colomnsArr);
        // var_dump($colomnsArr);
        foreach($colomnsArr as $field){
            if($i==1){
                $fieldsStr .= "   (   ";
            }

            if($field){
                // var_dump($field);
                $fieldsStr .= "   '{$field}'  ";
            }
            else{
                $fieldsStr .= "   ''  ";
            }
            
            if($i<$totalColomnsNums){
                $fieldsStr .= "   ,  ";
            }
            
            else{
                $fieldsStr .= "   )  ";
            }

            $i++;
        }
        return $fieldsStr;
}


~~~ 


## insertOnDuplocates

~~~ 
static function  insertOnDuplocates($Datas,$unqiueField){
        $mode = new self();

        $DatasBak = array_values($Datas);
        
        $Data = $DatasBak[0];
        
        $fieldsStr = \model\agreement\Order::transferColomnArrToStr(array_keys($Data));
        $valuesStr = \model\agreement\Order::transferDbArrsToStr($Datas);
         
        return  $mode->query("
                INSERT INTO 
                    ".self::$table."
                    {$fieldsStr}
                VALUES 
                    {$valuesStr}
                ON DUPLICATE KEY UPDATE 
                {$unqiueField} = {$unqiueField};

           ");
    } 

    static function transferColomnArrToStr($colomnsArr){
        $fieldsStr = "";
        $i = 1;
        $totalColomnsNums = count($colomnsArr);
        foreach($colomnsArr as $field){
            $fieldsStr .= "   `{$field}`  ";
            if($i<$totalColomnsNums){
                $fieldsStr .= "   ,  ";
            }

            $i++;
        }
        return $fieldsStr;
    }

    static function transferDbArrsToStr($colomnsArrs){
        $fieldsStr = "";
        $i = 1;
        $totalColomnsNums = count($colomnsArrs);
        // var_dump($colomnsArr);
        foreach($colomnsArrs as $valueItem){
            $fieldsStr .=  self::transferDbArrToStr($valueItem);

            
            if($i<$totalColomnsNums){
                $fieldsStr .= "   ,  ";
            }
            
            else{
                 
            }

            $i++;
        }
        return $fieldsStr;
    }

     static function transferDbArrToStr($colomnsArr){
        $fieldsStr = "";
        $i = 1;
        $totalColomnsNums = count($colomnsArr);
        // var_dump($colomnsArr);
        foreach($colomnsArr as $field){
            if($i==1){
                $fieldsStr .= "   (   ";
            }

            if($field){
                // var_dump($field);
                $fieldsStr .= "   '{$field}'  ";
            }
            else{
                $fieldsStr .= "   ''  ";
            }
            
            if($i<$totalColomnsNums){
                $fieldsStr .= "   ,  ";
            }
            
            else{
                $fieldsStr .= "   )  ";
            }

            $i++;
        }
        return $fieldsStr;
    }

~~~ 

## basic model actions
~~~ 
static function getMinId($table ='' ){
        if(empty($table)){
            $table =self::$table;
        }
        $model = new self();
        $resModel =$model->fetchOne("SELECT id FROM  ".$table);
        return  $resModel?$resModel->id:0;
    }

    static function getMaxId($table = ''){
        if(empty($table)){
            $table =self::$table;
        }
        $model = new self();
        $resModel =$model->fetchOne("SELECT max(id) as  id   FROM    ".$table);
        return  $resModel?$resModel->id:0;
    } 

    
    static function  addData($data){
        $model = new self();
        
        return  $model->add($data,true);

    }

    static function  updateData($data){
        $model = new self(); 
        return  $model->update($data,null,true);

    }


    static function getByIds($minId,$maxId,$field='*',$table= '' ){
        if(empty($table)){
            $table =self::$table;
        }
        $model = new self();
        return $model->fetchAll("SELECT {$field} from ".$table."
                    WHERE id >=  {$minId}
                    AND id <  {$maxId} 
                    
        "); 
    }


~~~ 

## export to csv  by  lines 

~~~ 
public function exportDb($request, $response)
    { 
         
        $filename = "reports.csv";
        header('Content-type: application/csv');
        header('Content-Disposition: attachment; filename='.$filename);
         

        $fp = fopen('php://output', 'w');
         
            
        $header = []; 
        fputcsv($fp, $header); 
        // 按page  取数据 取出后按行塞进csv 
        foreach ($rs as $data) {
                $fp = fopen('php://output', 'a');
                fputcsv($fp, $data);
                 
        }
        
        fclose($fp);


    }


~~~

## mysql  like  loacte 

~~~ 
可用到索引：
EXPLAIN SELECT name FROM company WHERE locate('123',name) >0 ;
用不到索引
EXPLAIN SELECT id,name FROM company WHERE locate('123',name) >0 ;

mysql> set profiling=1; 
mysql> SELECT name FROM company WHERE locate('北京',name) >0  order by id  desc  limit 5 ;
mysql> SELECT name FROM company WHERE name LIKE '%北京%'  order by id  desc  limit 5 ;
mysql> SHOW PROFILES; 
+----------+------------+-------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                               |
+----------+------------+-------------------------------------------------------------------------------------+
|        1 | 0.00051325 | SELECT name FROM company WHERE locate('北京',name) >0  order by id  desc  limit 5   |
|        2 | 0.00040625 | SELECT name FROM company WHERE name LIKE '%北京%'  order by id  desc  limit 5       |
+----------+------------+-------------------------------------------------------------------------------------+
mysql> show profile for query 1;
+--------------------------------+----------+
| Status                         | Duration |
+--------------------------------+----------+
| starting                       | 0.000034 |
| Waiting for query cache lock   | 0.000004 |
| starting                       | 0.000004 |
| checking query cache for query | 0.000081 |
| checking permissions           | 0.000006 |
| Opening tables                 | 0.000150 |
| init                           | 0.000026 |
| System lock                    | 0.000008 |
| Waiting for query cache lock   | 0.000003 |
| System lock                    | 0.000014 |
| optimizing                     | 0.000010 |
| statistics                     | 0.000017 |
| preparing                      | 0.000013 |
| Sorting result                 | 0.000005 |
| executing                      | 0.000004 |
| Sending data                   | 0.000061 |
| end                            | 0.000004 |
| query end                      | 0.000006 |
| closing tables                 | 0.000006 |
| freeing items                  | 0.000009 |
| Waiting for query cache lock   | 0.000003 |
| freeing items                  | 0.000019 |
| Waiting for query cache lock   | 0.000004 |
| freeing items                  | 0.000003 |
| storing result in query cache  | 0.000004 |
| cleaning up                    | 0.000017 |
+--------------------------------+----------+
mysql> show profile for query 2;
+--------------------------------+----------+
| Status                         | Duration |
+--------------------------------+----------+
| starting                       | 0.000063 |
| Waiting for query cache lock   | 0.000016 |
| starting                       | 0.000006 |
| checking query cache for query | 0.000062 |
| checking permissions           | 0.000017 |
| Opening tables                 | 0.000013 |
| init                           | 0.000023 |
| System lock                    | 0.000008 |
| Waiting for query cache lock   | 0.000003 |
| System lock                    | 0.000023 |
| optimizing                     | 0.000008 |
| statistics                     | 0.000016 |
| preparing                      | 0.000012 |
| Sorting result                 | 0.000004 |
| executing                      | 0.000004 |
| Sending data                   | 0.000058 |
| end                            | 0.000004 |
| query end                      | 0.000005 |
| closing tables                 | 0.000007 |
| freeing items                  | 0.000008 |
| Waiting for query cache lock   | 0.000003 |
| freeing items                  | 0.000018 |
| Waiting for query cache lock   | 0.000003 |
| freeing items                  | 0.000003 |
| storing result in query cache  | 0.000004 |
| cleaning up                    | 0.000019 |
+--------------------------------+----------+


~~~


## qucik count 
~~~ 

SLOW:

EXPLAIN SELECT COUNT(*) FROM company WHERE id > 5 ;

QUICK :
EXPLAIN  SELECT (SELECT COUNT(*) FROM company) - COUNT(*) FROM company WHERE id <= 1 ;

~~~ 

## ROLLUP
~~~ 

SELECT agreement_id, SUM(money) FROM agreements__paylog GROUP by agreement_id WITH ROLLUP ;

~~~ 


## SUM IF  
~~~ 
SELECT SUM(IF(deal_status=1,1,0)),SUM(IF(deal_status=2,1,0)),SUM(IF(deal_status=3,1,0)) FROM company ;
SELECT 
	 
	SUM(
		IF(
			companySize < 100 
			AND socialSecurityNumber < 100 
			AND province = '陕西省', 
			1, 
			0
		)
	) as 'less than 100 陕西省', 
	SUM(
		IF(
			(
				(
					companySize >= 100 
					AND companySize < 300
				) 
				OR (
					socialSecurityNumber >= 100 
					AND socialSecurityNumber < 300
				)
			) 
			AND province = '陕西省', 
			1, 
			0
		)
	) as '100-300 陕西省', 
	SUM(
		IF(
			(
				(
					companySize >= 300 
					AND companySize < 500
				) 
				OR (
					socialSecurityNumber >= 300 
					AND socialSecurityNumber < 500
				)
			) 
			AND province = '陕西省', 
			1, 
			0
		)
	) as '300-500 陕西省', 
	SUM(
		IF(
			(
				(
					companySize >= 500 
					AND companySize < 1000
				) 
				OR (
					socialSecurityNumber >= 500 
					AND socialSecurityNumber < 1000
				)
			) 
			AND province = '陕西省', 
			1, 
			0
		)
	) as '500-1000 陕西省', 
	SUM(
		IF(
			(
				(
					companySize >= 1000 
					AND companySize < 3000
				) 
				OR (
					socialSecurityNumber >= 1000 
					AND socialSecurityNumber < 3000
				)
			) 
			AND province = '陕西省', 
			1, 
			0
		)
	) as '1000-3000 陕西省', 
	SUM(
		IF(
			(
				companySize >= 3000 
				OR socialSecurityNumber >= 3000
			) 
			AND province = '陕西省', 
			1, 
			0
		)
	) as ' 3000 plus  陕西省', 
	-- split 陕西省
	SUM(
		IF(
			companySize < 100 
			AND socialSecurityNumber < 100 
			AND city = '济南市', 
			1, 
			0
		)
	) as 'less than 100 济南市', 
	SUM(
		IF(
			(
				(
					companySize >= 100 
					AND companySize < 300
				) 
				OR (
					socialSecurityNumber >= 100 
					AND socialSecurityNumber < 300
				)
			) 
			AND city = '济南市', 
			1, 
			0
		)
	) as '100-300 济南市', 
	SUM(
		IF(
			(
				(
					companySize >= 300 
					AND companySize < 500
				) 
				OR (
					socialSecurityNumber >= 300 
					AND socialSecurityNumber < 500
				)
			) 
			AND city = '济南市', 
			1, 
			0
		)
	) as '300-500 济南市', 
	SUM(
		IF(
			(
				(
					companySize >= 500 
					AND companySize < 1000
				) 
				OR (
					socialSecurityNumber >= 500 
					AND socialSecurityNumber < 1000
				)
			) 
			AND city = '济南市', 
			1, 
			0
		)
	) as '500-1000 济南市', 
	SUM(
		IF(
			(
				(
					companySize >= 1000 
					AND companySize < 3000
				) 
				OR (
					socialSecurityNumber >= 1000 
					AND socialSecurityNumber < 3000
				)
			) 
			AND city = '济南市', 
			1, 
			0
		)
	) as '1000-3000 济南市', 
	SUM(
		IF(
			(
				companySize >= 3000 
				OR socialSecurityNumber >= 3000
			) 
			AND city = '济南市', 
			1, 
			0
		)
	) as ' 3000 plus  济南市'  
FROM 
	company__third_party_clues_base_info 
WHERE 
	id >= 1 
	AND id <= 3000000 ;


~~~ 


## quick  limit large offset  

~~~ 
SLOW:
select 
	clue.* 
from 
	company__third_party_clues_base_info as clue  FORCE INDEX (socialSecrity)
where 
	1 
	AND clue.`deal_status` in (0, 1) 
order by 
	clue.socialSecurityNumber desc 
LIMIT 
	2750000, 20 ;


QUCIK:
SELECT 
	l.* 
FROM 
	(
		SELECT 
			clue.id 
		FROM 
			company__third_party_clues_base_info as clue   
		where 
			1 
			AND clue.`deal_status` in (0, 1) 
		ORDER BY 
			clue.socialSecurityNumber 
		LIMIT 
			2650000, 20
	) o 
	JOIN company__third_party_clues_base_info l ON l.id = o.id 
ORDER BY 
	l.socialSecurityNumber ;    



mysql>  SHOW  PROFILES;                                                                                                                                                                     
 mysql> SHOW PROFILES;  
+----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Query_ID | Duration    | Query                                                                                                                                                                                                                                                                                 |
+----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|        1 | 18.07380375 | select clue.* from company__third_party_clues_base_info as clue FORCE INDEX (socialSecrity) where 1 AND clue.`deal_status` in (0, 1) order by clue.socialSecurityNumber desc LIMIT 2700000,10                                                                                         |
|        2 |  3.07341700 | SELECT l.* FROM ( SELECT clue.id FROM company__third_party_clues_base_info as clue where 1 AND clue.`deal_status` in (0, 1) ORDER BY clue.socialSecurityNumber LIMIT 2660000, 20 ) o JOIN company__third_party_clues_base_info l ON l.id = o.id ORDER BY l.socialSecurityNumber  desc |
+----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

mysql> show profile for query 1;
+--------------------------------+-----------+
| Status                         | Duration  |
+--------------------------------+-----------+
| starting                       |  0.000032 |
| Waiting for query cache lock   |  0.000006 |
| starting                       |  0.000004 |
| checking query cache for query |  0.000116 |
| checking permissions           |  0.000009 |
| Opening tables                 |  0.000165 |
| init                           |  0.000063 |
| System lock                    |  0.000010 |
| Waiting for query cache lock   |  0.000004 |
| System lock                    |  0.000018 |
| optimizing                     |  0.000012 |
| statistics                     |  0.000017 |
| preparing                      |  0.000013 |
| Sorting result                 |  0.000005 |
| executing                      |  0.000004 |
| Sending data                   | 18.073170 |
| end                            |  0.000013 |
| query end                      |  0.000011 |
| closing tables                 |  0.000011 |
| freeing items                  |  0.000048 |
| Waiting for query cache lock   |  0.000003 |
| freeing items                  |  0.000023 |
| Waiting for query cache lock   |  0.000003 |
| freeing items                  |  0.000003 |
| storing result in query cache  |  0.000004 |
| logging slow query             |  0.000030 |
| cleaning up                    |  0.000011 |
+--------------------------------+-----------+
27 rows in set, 1 warning (0.00 sec)

mysql> show profile for query 2;
+--------------------------------+----------+
| Status                         | Duration |
+--------------------------------+----------+
| starting                       | 0.000041 |
| Waiting for query cache lock   | 0.000077 |
| starting                       | 0.000013 |
| checking query cache for query | 0.000093 |
| checking permissions           | 0.000004 |
| checking permissions           | 0.000006 |
| Opening tables                 | 0.000131 |
| init                           | 0.000087 |
| System lock                    | 0.000009 |
| Waiting for query cache lock   | 0.000003 |
| System lock                    | 0.000017 |
| optimizing                     | 0.000005 |
| optimizing                     | 0.000010 |
| statistics                     | 0.000051 |
| preparing                      | 0.000013 |
| Sorting result                 | 0.000007 |
| statistics                     | 0.000041 |
| preparing                      | 0.000008 |
| Creating tmp table             | 0.000042 |
| Sorting result                 | 0.000007 |
| executing                      | 0.000008 |
| Sending data                   | 0.000027 |
| executing                      | 0.000003 |
| Sending data                   | 0.000004 |
| Creating sort index            | 3.072468 |
| Creating sort index            | 0.000105 |
| end                            | 0.000006 |
| query end                      | 0.000009 |
| removing tmp table             | 0.000010 |
| query end                      | 0.000007 |
| closing tables                 | 0.000004 |
| removing tmp table             | 0.000006 |
| closing tables                 | 0.000010 |
| freeing items                  | 0.000021 |
| Waiting for query cache lock   | 0.000003 |
| freeing items                  | 0.000029 |
| Waiting for query cache lock   | 0.000004 |
| freeing items                  | 0.000004 |
| storing result in query cache  | 0.000005 |
| cleaning up                    | 0.000024 |
+--------------------------------+----------+



~~~    


## SHOW   FULL  PROCESSLIST  
~~~ 
SHOW   FULL  PROCESSLIST ;
~~~ 


##  查看 sleep 
~~~
注意：有可能是已经执行完的：等待发起新的请求前最后一次的sql
SELECT 
	b.processlist_id, 
	c.db, 
	a.sql_text, 
	c.command, 
	c.time, 
	c.state 
FROM 
	performance_schema.events_statements_current a 
	JOIN performance_schema.threads b USING(thread_id)  
	JOIN information_schema.processlist c ON b.processlist_id = c.id ;


mysql> select max(id) from  debug ;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    2673
Current database: aibangmang

+----------+
| max(id)  |
+----------+
| 11921124 |
+----------+
1 row in set (1.49 sec)

mysql> SELECT b.processlist_id, c.db, a.sql_text, c.command, c.time, c.state FROM performance_schema.events_statements_current a JOIN performance_schema.threads b USING(thread_id)   JOIN information_schema.processlist c ON b.processlist_id = c.id LIMIT 0, 25;
+----------------+------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+------+-----------+
| processlist_id | db         | sql_text                                                                                                                                                                                                                                                    | command | time | state     |
+----------------+------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+------+-----------+
|           2673 | aibangmang | select max(id) from  debug                                                                                                                                                                                                                                  | Sleep   |  119 |           |
|           2677 | aibangmang | SELECT b.processlist_id, c.db, a.sql_text, c.command, c.time, c.state FROM performance_schema.events_statements_current a JOIN performance_schema.threads b USING(thread_id)   JOIN information_schema.processlist c ON b.processlist_id = c.id LIMIT 0, 25 | Query   |    0 | executing |
+----------------+------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+------+-----------+
 


~~~

## mysql  table size 
~~~

SELECT
TABLE_NAME,
table_rows,
data_length,
index_length,
round(
(
(data_length + index_length) / 1024 / 1024
),
2
) "Size in MB"
FROM
information_schema.TABLES
WHERE
table_schema = "aibangmang"
ORDER BY
(data_length + index_length) DESC ;


SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'aibangmang' and TABLE_TYPE='BASE TABLE';

~~~

## binlog

~~~ 

show VARIABLES LIKE '%binlog%' ;

binlog_rows_query_log_events : 设置为 ON 可以看到执行的sql

SHOW MASTER LOGS ;

show binlog events in 'mysql-bin.000236' ; 
show binlog events in 'mysql-bin.000001'  limit 1000 ;

info 里是实际执行的sql
【config 需要设置为 binlog_format=mixed  才会展示执行的sql】

~~~ 


## 临时查看 所有执行的sql
~~~

SHOW VARIABLES LIKE '%general_log%' ;


SET GLOBAL log_output = 'TABLE';SET GLOBAL general_log = 'ON';  //日志开启

SET GLOBAL log_output = 'TABLE'; SET GLOBAL general_log = 'OFF';  //日志关闭 

SELECT * from mysql.general_log ORDER BY event_time DESC ;

如果嫌占用空间，查询完毕及时关了 

~~~

## mysql json

~~~
SELECT raw_data->'$.user' as ruser,raw_data->'$.agentsNo' as ragentNo,raw_data FROM `outbound__tell_receiving` WHERE id = 4000000;

SELECT  ROUND(trace->'$.callee')  FROM outbound__abnormal_data_trace WHERE description='接收标记结果' ORDER by id desc LIMIT 2

~~~

##  mysql slave 

~~~
slave mysql:
show slave status  ;

Seconds_Behind_Master: NULL异常 0 延时不存在
以下两项同时为YES 才是正常主从复制
Slave_IO_Running ：
Slave_SQL_Running ： 

一项为NO 表示异常：
查看错误：
Last_Error：
详细错误信息  

stop slave ;
set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
start slave ;



show  slave  status ;

Last_SQL_Error: Relay log read failure: Could not parse relay log event entry. 
The possible reasons are: the master's binary log is corrupted (
you can check this by running 'mysqlbinlog' on the binary log), 
the slave's relay log is corrupted (you can check this by running 'mysqlbinlog' 
on the relay log), a network problem, or a bug in the master's or slave's MySQL code. 
If you want to check the master's binary log or slave's relay log, 
you will be able to know their names by issuing 'SHOW SLAVE STATUS' on this slave.

show  slave  status \G;
Relay_Master_Log_File:  binlog.000021     //slave库已读取的master的binlog
Exec_Master_Log_Pos: 676992028           //在slave上已经执行的position位置点

stop slave;
change master to master_log_file='binlog.000021' , master_log_pos=676992028;
start slave;


可读格式展示
 sudo mysqlbinlog /var/lib/mysql/VM-0-4-ubuntu-bin.000002 --start-position 74401248 --stop-position 74401286    -v --base64-output=DECODE-ROWS


 sudo mysqlbinlog /var/lib/mysql/VM-0-4-ubuntu-bin.000002    -v --base64-output=DECODE-ROWS



/etc/init.d/mysql start
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
show variables like 'datadir';



mysql> show binlog events in 'VM-0-4-ubuntu-bin.000004' from  418  limit  10 ;
+--------------------------+-----+------------+-----------+-------------+---------------------------------+
| Log_name                 | Pos | Event_type | Server_id | End_log_pos | Info                            |
+--------------------------+-----+------------+-----------+-------------+---------------------------------+
| VM-0-4-ubuntu-bin.000004 | 418 | Write_rows |         0 |         467 | table_id: 110                   |
| VM-0-4-ubuntu-bin.000004 | 467 | Write_rows |         0 |         527 | table_id: 112 flags: STMT_END_F |
| VM-0-4-ubuntu-bin.000004 | 527 | Xid        |         0 |         558 | COMMIT /* xid=237 */            |
| VM-0-4-ubuntu-bin.000004 | 558 | Rotate     |         0 |         613 | VM-0-4-ubuntu-bin.000005;pos=4  |

如果不可见具体的sql

sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf 
binlog_format=mixed

mysql> show binlog events in 'VM-0-4-ubuntu-bin.000007' ;
+--------------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| Log_name                 | Pos | Event_type     | Server_id | End_log_pos | Info                                                                    |
+--------------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------------+
| VM-0-4-ubuntu-bin.000007 |   4 | Format_desc    |         0 |         123 | Server ver: 5.7.32-0ubuntu0.16.04.1-log, Binlog ver: 4                  |
| VM-0-4-ubuntu-bin.000007 | 123 | Previous_gtids |         0 |         154 |                                                                         |
| VM-0-4-ubuntu-bin.000007 | 154 | Anonymous_Gtid |         0 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                    |
| VM-0-4-ubuntu-bin.000007 | 219 | Query          |         0 |         306 | BEGIN                                                                   |
| VM-0-4-ubuntu-bin.000007 | 306 | Query          |         0 |         447 | use `test`; insert into `data` values (101,'2020-01-01 10:00:00','1',1) |
| VM-0-4-ubuntu-bin.000007 | 447 | Xid            |         0 |         478 | COMMIT /* xid=21 */                 




~~~
 


## VIEW  view
~~~

CREATE VIEW Customers
AS
SELECT
order_id,id FROM agreements__paylog GROUP by order_id; 

CREATE OR  REPLACE  VIEW;

 SHOW CREATE VIEW `Customers`; 
 
~~~

## WITh 
~~~
WITH  tab_1 AS (
	SELECT 
		company_id, 
		min(sign_date) 
	FROM 
		agreements__order 
	GROUP by 
		company_id
)

SELECT  * FROM tab_1


~~~


## quick delete 
~~~
DELETE QUICK  FROM  table ;

QUICK  & JOIN  (结合使用)

CREATE TABLE `tmp_ids` (
	`id` int(11) NOT NULL AUTO_INCREMENT, 
	PRIMARY KEY (`id`)
) ENGINE = MyISAM AUTO_INCREMENT = 36768 DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci COMMENT = '新建表模板' ;

INSERT into tmp_ids 
SELECT 
	id 
FROM 
	`outbound__tell_result` 
WHERE 
	id >= 27767 
	AND id <= 30767;

    DELETE QUICK tell 
FROM 
	outbound__tell_result as tell 
	JOIN tmp_ids as id2 on tell.id = id2.id ;


~~~


## git  revert 

~~~

git revert 74fd458 ;

1:删除某个提交  
2:错误的合并了某个分支： 错误合并分支之后，还进行了正常的提交  想删除那次错误的合并，并保留之后的提交

My@My-THINK MINGW64 /e/phpprojects/phpExamples (master)
$ git revert  104010a231c52a0169c62ae5488e4856d69cddd7
[master df2b9c7] Revert "add 01"
 1 file changed, 1 deletion(-)
 delete mode 100644 README - 01.md

~~~


## grep split large log 
~~~
扎到满足条件的开头位置
cat  debug_db_20200828.log    | tail -n +1720000 | head -n 100  

扎到满足条件的结束位置
cat  debug_db_20200828.log    | tail -n +1920000 | head -n 100

cat  debug_db_20200828.log    | tail -n +1720000 | head -n 200000 > tmp_0828.log

压缩
gzip tmp_0828.log

下载
sz -y tmp_0828.log.gz

~~~


## grep awk 
~~~

cat tmp_sql.log  |grep "SELECT" | awk '{print $3}' |sort -rn |head -5 

Top 10 IP Addresses
awk '{ print $1}' access.log.2016-05-08 | sort | uniq -c | sort -nr | head -n 10
cat access.log | awk '{print $1}' | sort -n | uniq -c | sort -nr | head -10
~~~
 

## nslook  IP address
~~~

nslookup  XXX.org
XXX.org
Address:  101.200.90.180
 
docker03 172.17.1.79 [WORK]$curl ipinfo.io/ip
101.200.90.180

curl api.ipify.org

public ip :

curl ifconfig.me

curl -4/-6 icanhazip.com

curl ipinfo.io/ip

curl api.ipify.org

curl checkip.dyndns.org

dig +short myip.opendns.com @resolver1.opendns.com

host myip.opendns.com resolver1.opendns.com

curl ident.me

curl bot.whatismyipaddress.com

curl ipecho.net/plain


private Ip:

ifconfig -a

ip addr (ip a)

hostname -I | awk '{print $1}'

ip route get 1.2.3.4 | awk '{print $7}' 

~~~


## top

~~~
top 
c 最占用性能的程序 （比如负载高 c之后全是mysql  说明是mysql的问题）
1  几核

load avg  :  <1 (或者2)  单核
             <1*4 (或者2*4)  4核
             <1*8 (或者2*8)  

注意 即使负载相对低 但是c之后全是一个程序 也是问题 比如 全是apache apache很可能会堵住

top -c -n 1

ps -f -p 11818 


top c 

一堆 apache 或者 nginx  ：apache 拥堵

keyword 关键字：What the full process thats running?
~~~

## df du  memeory
~~~
磁盘占用
new03 172.17.22.202 [var]$df -h

new03 172.17.22.202 [var]$df -a

文件大小（全部）
new03 172.17.22.202 [var]$du -sh 
每个文件的大小
new03 172.17.22.202 [var]$du -h * 

find ./ -type f -size +1G
find Downloads/ -type f -size +4G

查找文件夹
find /WORK -name "udf"

找到子文件里包含gif的文件 删除

 find   . -name '*.gif'  -delete

~~~


## method_exists
~~~

if(method_exists('xx\apiExample', $methodName_checkParams)){
}

~~~


##  preg_match

~~~

//取得指定位址的內容，並儲存至text
$str=file_get_contents('https://mp.weixin.qq.com/s?__biz=MjM5NTE1OTQyMQ==&mid=2650784873&idx=1&sn=95f9acb28ee67a66a36f12873e813f4d&chksm=bef7a87b8980216dbc6bf93590f00edad6d3fcc612bb98678156ca7ee26e6eb2e46160b22161&scene=27#wechat_redirect');

// <span class="profile_meta_value">Beijing_Daily</span>
preg_match('/<span.+class=\"?profile_meta_value\"?>*.+span>/u', $str, $match_name);

var_dump($match_name);
print_r($match_name[0]);


~~~

## CallAPI 
~~~

public  function CallAPIV2($method, $url, $data,$headers,$isHttps=false,$ifDebug=false) {
        ob_start();
        $out = fopen('php://output', 'w');

        $curl = curl_init($url);
        curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);

        curl_setopt($curl, CURLOPT_VERBOSE, 1);
        curl_setopt($curl, CURLOPT_STDERR, $out);
        if($ifDebug){
            curl_setopt($curl, CURLOPT_HEADER, 1);
        }

        switch ($method) {
            case "GET":
                curl_setopt($curl, CURLOPT_POSTFIELDS, json_encode($data));
                curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "GET");
                break;
            case "POST":
                curl_setopt($curl, CURLOPT_POSTFIELDS, json_encode($data));
                curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "POST");
                break;
            case "PUT":
                curl_setopt($curl, CURLOPT_POSTFIELDS, json_encode($data));
                curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "PUT");
                break;
            case "DELETE":
                curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "DELETE");
                curl_setopt($curl, CURLOPT_POSTFIELDS, json_encode($data));
                break;
        }
        curl_setopt($curl, CURLOPT_HTTPHEADER, $headers);
        if($isHttps){
            //这个是重点,规避ssl的证书检查。
            curl_setopt($curl, CURLOPT_SSL_VERIFYPEER, false);
            // 跳过host验证
            curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, FALSE);
        }

        //curl_setopt($curl,CURLINFO_HEADER_OUT,true);


        $response = curl_exec($curl);
        fclose($out);
        $debug = ob_get_clean();

        $this->raw_response = $response;

        if($ifDebug){
            var_dump($debug);
            echo ("Request-Body(URL Encoded):\r\n" . http_build_query($data) . "\r\n").'<br/>';
            echo ("Request-Body(Json Encoded):\r\n" . json_encode($data) . "\r\n");
            var_dump("Response:\r\n" . $response . "\r\n");
            var_dump(curl_getinfo($curl)) ;
        }

        // \model\DebugSimple::addData('请求结果',json_encode([
        //     'Request-Header'=>$debug,
        //     'Request-Body(URL Encoded)'=>http_build_query($data),
        //     'Request-Body(Json Encoded)'=>json_encode($data) ,
        //     'Response'=>$response ,
        //     'Response-info'=>json_encode(curl_getinfo($curl)) ,
        // ]));

        $data = json_decode($response,true);

        /* Check for 404 (file not found). */
        $httpCode = curl_getinfo($curl, CURLINFO_HTTP_CODE);

        // Check the HTTP Status code
        switch ($httpCode) {
            case 200:
                $error_status = "200: Success";
                \yoka\Debug::log('请求成功 返回结果',[
                    $error_status,
                    $response,
                    $response->data,
                    $response['data'],
                    $response->code,
                    $data,
                ]);
                return ($data);
                break;
            case 404:
                $error_status = "404: API Not found";
                break;
            case 500:
                $error_status = "500: servers replied with an error.";
                break;
            case 502:
                $error_status = "502: servers may be down or being upgraded. Hopefully they'll be OK soon!";
                break;
            case 503:
                $error_status = "503: service unavailable. Hopefully they'll be OK soon!";
                break;
            default:
                $error_status = "Undocumented error: " . $httpCode . " : " . curl_error($curl);
                break;
        }
        curl_close($curl);
        echo $error_status;
        die;
    }

~~~

## Adapter

~~~

interface SmsInterface_{public function sendSms($mobile,$text,$tplName);}

class SmsAdapterOne implements SmsInterface_ {
    public function sendSms($mobile,$text,$tplName){
    }
}

class SmsAdapterTwo implements SmsInterface_ {
    public function sendSms($mobile,$text,$tplName){
    }
}

class sms {
    private static $sms = null;
    private static $defaultAdapaer = 'channel_one';

    public static $config = [
            'channel_one'=>[
                'name'=>'渠道一',
                'adaperClass'=>'\model\Sms\SmsAdapterOne',
            ],
            'channel_two'=>[
            'name'=>'渠道二',
            'adaperClass'=>'\model\Sms\SmsAdapterOne',
        ],
    ];

    public static function getInstance($channel)
    {
        if(!$channel){
            $channel = self::$defaultAdapaer;
        }
        if(!self::$sms){
            self::$sms = new self::$config[$channel]['adaperClass']();
        }
        return self::$sms;
    }

    public function __construct($channel='')
    {
        self::getInstance($channel);
    }

    public function send($mobile,$text){
        self::$sms->sendSms($mobile,$text);
    }

}


$sms = new sms();
$sms->send('','');

~~~

## debug_backtrace()
~~~
\model\DebugSimple::addData('sss',' '.json_encode2(debug_backtrace()));

~~~

## operate__log

~~~



CREATE TABLE `operate__log` (
	`id` int(11) NOT NULL AUTO_INCREMENT, 
	`unique_key_raw` varchar(80) NOT NULL, 
	`unique_key` varchar(80) NOT NULL DEFAULT '', 
	`old_content` text NOT NULL, 
	`new_content` text NOT NULL, 
	`changed_content` text NOT NULL, 
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	`remark` text CHARACTER SET utf8 COLLATE utf8_unicode_ci NOT NULL, 
	`update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, 
	`index_name` varchar(80) NOT NULL DEFAULT '', 
	`table_name` varchar(32) NOT NULL DEFAULT '', 
	`state` tinyint(1) NOT NULL DEFAULT '0', 
	`operator` varchar(16) NOT NULL, 
	PRIMARY KEY (`id`), 
	KEY `unique` (`unique_key`, `table_name`)
) ENGINE = MyISAM AUTO_INCREMENT = 18917 DEFAULT CHARSET = utf8 COMMENT = '操作日志表'


static  $config_fields = [
        'company'=>[
            'name',
            'sales_id_rollin', 
            'sales_id',
            'sales_id_rollin', 
            'state', 
            'can_mail',
            'local_province',
            'local_city',
            'deal_status',
            'paycard_allow',
        ], 
        'company__contact'=>[
            'company_id',
            // 'sales_id',
            'mobile',
            'email'
        ], 

    ];

static function  saveOperateLog($infos,$datasInfo){
        
       if(
           empty($infos['index_name'])||
           empty($datasInfo['new_data'])||
           empty($infos['table'])||
           empty($datasInfo['old_data']['id'])
       ){
           return \yoka\YsError::error('保存日志 参数缺失1',$datasInfo['new_data']);
       }

        empty($infos['operator']) && $infos['operator'] = 'system';

        #### 1.读取配置 config
        if(
            empty(self::$config_fields[$infos['table']])
        )
        {
            return \yoka\YsError::error('保存日志 参数缺失2',$datasInfo['new_data']);
        }

        #### 2 生成key
        $model = new self();

        //    $uniquekeyRes = self::getUniqueKey($infos['table'],$datasInfo['new_data'],$datasInfo['old_data']['id']);
        //    if(!$uniquekeyRes){
        //        return \yoka\YsError::error('获取unique key失败');;
        //    }

        //    $oldModel = $model->fetchOne([
        //        ####  md5 值
        //        'unique_key'=>$uniquekeyRes['unique_key'],
        //        'table_name'=>$infos['table'],
        //        'state'=>0,
        //    ]);

        //    #### 木有变更内容
        //    if($oldModel){
        //        return true;
        //    }

        //---3 有变更内容
        $changeDatas = self::getChangeDatas(
            $datasInfo['old_data'],
            $datasInfo['new_data'],
            self::$config_fields[$infos['table']]
        );
        if(empty($changeDatas)){
            \yoka\Debug::log('no change,continue  ',[]);
             return  true;
        }

        return $model->add([
            'unique_key'=>$infos['index_name']  ,
            'unique_key_raw'=>$infos['index_name'] ,
            // 'old_content'=>json_encode2($datasInfo['old_data']),
            // 'new_content'=>json_encode2($datasInfo['new_data']),
            'changed_content'=>json_encode2($changeDatas),
            'index_name'=>$infos['index_name'] ,
            'table_name'=>$infos['table'],
            'operator'=>$infos['operator'],
            'remark'=>$infos['remark'],
        ]);

    }

    static function getChangeDatas($oldInfo,$newInfo,$fields){
        $changeContent = [];
        foreach ($fields as $field){
            //---要更新的data设置了该key时
            if(isset($newInfo[$field])){
                if($newInfo[$field]!=$oldInfo[$field]){
                    $changeContent[$field]=[
                        'old'=>$oldInfo[$field],
                        'new'=>$newInfo[$field],
                    ];
                }
            }
        }
        return $changeContent;
    }

static function checkIfNeedsSaveLogByTable($tableName){
        if(
            in_array($tableName,array_keys(\model\OperateLog::$config_fields))
        ){
            return true;
        }
        else{
            return  false;
        }
					
    }
    static function checkIfNeedsSaveLogByDbInfo($tableName,$dbData){
        $fieldsArr = \model\OperateLog::$config_fields[$tableName];
        $changedFieldsArr = array_keys($dbData);

        $intersectArr = array_intersect($fieldsArr, $changedFieldsArr);
        if(!empty($intersectArr)){
            return true;
        }
        else{
            return  false;
        }
					
    }

~~~

## execute after response 

~~~ 
	public function respondOK($text = null)
	{
		// check if fastcgi_finish_request is callable
		if (is_callable('fastcgi_finish_request')) {
			if ($text !== null) {
				echo $text;
			}
			/*
			 * http://stackoverflow.com/a/38918192
			 * This works in Nginx but the next approach not
			 */
			session_write_close();
			fastcgi_finish_request();
	 
			return;
		}
	 
		ignore_user_abort(true);
	 
		ob_start();
	 
		if ($text !== null) {
			echo $text;
		}
	 
		$serverProtocol = filter_input(INPUT_SERVER, 'SERVER_PROTOCOL', FILTER_SANITIZE_STRING);
		header($serverProtocol . ' 200 OK');
		// Disable compression (in case content length is compressed).
		header('Content-Encoding: none');
		header('Content-Length: ' . ob_get_length());
	 
		// Close the connection.
		header('Connection: close');
	 
		ob_end_flush();
		ob_flush();
		flush();
	}

~~~

## chown
~~~
new03 172.17.22.202 [aibangmang]$l |grep service__income
-rw-r----- 1 root  root   8.9K 2020-08-27 10:49:52 service__income_month.frm
-rw-r----- 1 root  root   761K 2020-08-27 10:49:52 service__income_month.MYD
-rw-r----- 1 root  root   130K 2020-08-27 10:49:52 service__income_month.MYI

new03 172.17.22.202 [aibangmang]$chown mysql  service__income*
new03 172.17.22.202 [aibangmang]$l |grep service__income
-rw-r----- 1 mysql root   8.9K 2020-08-27 10:49:52 service__income_month.frm
-rw-r----- 1 mysql root   761K 2020-08-27 10:49:52 service__income_month.MYD
-rw-r----- 1 mysql root   130K 2020-08-27 10:49:52 service__income_month.MYI

new03 172.17.22.202 [aibangmang]$chown mysql:mysql  service__income*
new03 172.17.22.202 [aibangmang]$l |grep service__income
-rw-r----- 1 mysql mysql  8.9K 2020-08-27 10:49:52 service__income_month.frm
-rw-r----- 1 mysql mysql  761K 2020-08-27 10:49:52 service__income_month.MYD
-rw-r----- 1 mysql mysql  130K 2020-08-27 10:49:52 service__income_month.MYI


~~~

## php crawl website data
~~~
php parse html class  

$some_link = 'https://xue.youdao.com/';
$tagName = 'div';
$attrName = 'class';
$attrValue = 'content-article';

$dom = new DOMDocument;
$dom->preserveWhiteSpace = false;
@$dom->loadHTMLFile($some_link); 
$html = getTags( $dom, $tagName, $attrName, $attrValue );
echo $html;

function getTags( $dom, $tagName, $attrName, $attrValue ){
    $html = '';
    $domxpath = new DOMXPath($dom);
    $newDom = new DOMDocument;
    $newDom->formatOutput = true;

    $filtered = $domxpath->query("//$tagName" . '[@' . $attrName . "='$attrValue']");
    // $filtered =  $domxpath->query('//div[@class="className"]');
    // '//' when you don't know 'absolute' path

    // since above returns DomNodeList Object
    // I use following routine to convert it to string(html); copied it from someone's post in this site. Thank you.
    $i = 0;
    while( $myItem = $filtered->item($i++) ){
        $node = $newDom->importNode( $myItem, true );    // import node
        $newDom->appendChild($node);                    // append node
    }
    $html = ($newDom->saveHTML());
    return $html;
}

UTF-8 :

$some_link = 'https://xue.youdao.com/';
$profile = file_get_contents($some_link);


// $profile = '<p>イリノイ州シカゴにて、アイルランド系の家庭に、9</p>';
$dom = new DOMDocument();
$dom->loadHTML(mb_convert_encoding($profile, 'HTML-ENTITIES', 'UTF-8'));

// echo $dom->saveHTML();

$tags = $dom->getElementsByTagName('img');

foreach ($tags as $tag) {
    var_dump($tag);
    //    echo $tag->getAttribute('href').' | '.$tag->nodeValue."\n";
}



~~~


## Extract numbers from a string
~~~


$str = 'In My Cart : 11 items';
$int = (int) filter_var($str, FILTER_SANITIZE_NUMBER_INT);



~~~

## PARTITIONS   EXAMPLE
~~~
分区示例
 CREATE TABLE `company__third_party_clues_base_info` (
	`id` int(11) NOT NULL AUTO_INCREMENT, 
	`third_party_clues_id` int(11) DEFAULT NULL, 
	`third_party_id` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`companyName` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`socialSecurityNumber` int(11) NOT NULL DEFAULT '0', 
	`company_id` int(11) NOT NULL DEFAULT '0', 
	`deal_status` int(6) NOT NULL DEFAULT '0', 
	`companySize` varchar(16) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`entType` varchar(32) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`officialweb` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`city` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`city_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`regCapital` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`arInsuranceNumber` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`legalPerson` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`regDate` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`creditNo` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAddr` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAuthority` varchar(80) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry_id` int(10) NOT NULL DEFAULT '0', 
	`openStatus` varchar(64) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`related_company_nums` int(11) NOT NULL DEFAULT '0', 
	`related_contacts_nums` int(11) NOT NULL DEFAULT '0', 
	`related_companys_emplyee_nums` int(11) NOT NULL DEFAULT '0', 
	`state` int(11) NOT NULL, 
	`urlRecords` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	`update_time` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP, 
	`remark` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`del_flag` int(11) NOT NULL, 
	PRIMARY KEY (`id`), 
	UNIQUE KEY `thirds_ids` (`third_party_id`), 
	KEY `third_party_clues_id` (`third_party_clues_id`), 
	KEY `companyName` (`companyName`), 
	KEY `socialSecrity` (`socialSecurityNumber`), 
	KEY `related_company_nums` (`related_company_nums`), 
	KEY `deal_status` (`deal_status`), 
	KEY `company_id` (
		`related_companys_emplyee_nums`
	) USING BTREE
) ENGINE = MyISAM AUTO_INCREMENT = 4186208 DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci COMMENT = '第三方线索表基础信息表' 


CREATE TABLE `company__third_party_clues_base_info2` (
	`id` int(11) NOT NULL AUTO_INCREMENT ,
    `third_party_clues_id` int(11) DEFAULT NULL, 
	`third_party_id` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`companyName` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`socialSecurityNumber` int(11) NOT NULL DEFAULT '0', 
	`company_id` int(11) NOT NULL DEFAULT '0', 
	`deal_status` int(6) NOT NULL DEFAULT '0', 
	`companySize` int(11) NOT NULL DEFAULT '0', 
	`entType` varchar(32) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`officialweb` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`city` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`city_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`regCapital` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`arInsuranceNumber` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`legalPerson` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`regDate` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`creditNo` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAddr` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAuthority` varchar(80) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry_id` int(10) NOT NULL DEFAULT '0', 
	`openStatus` varchar(64) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`related_company_nums` int(11) NOT NULL DEFAULT '0', 
	`related_contacts_nums` int(11) NOT NULL DEFAULT '0', 
	`related_companys_emplyee_nums` int(11) NOT NULL DEFAULT '0', 
	`state` int(11) NOT NULL, 
	`urlRecords` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	`update_time` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP, 
	`remark` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`del_flag` int(11) NOT NULL, 
	PRIMARY KEY (
		`id`, deal_status, companySize, socialSecurityNumber
	), 
	KEY `thirds_ids` (`third_party_id`), 
	KEY `third_party_clues_id` (`third_party_clues_id`), 
	KEY `companyName` (`companyName`), 
	KEY `socialSecrity` (`socialSecurityNumber`), 
	KEY `related_company_nums` (`related_company_nums`), 
	KEY `deal_status` (`deal_status`), 
	KEY `company_id` (
		`related_companys_emplyee_nums`
	) USING BTREE
) ENGINE = MyISAM DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci PARTITION BY RANGE COLUMNS(
	deal_status, companySize, socialSecurityNumber
) (
	PARTITION p0 
	VALUES 
		LESS THAN (1, 1, 1), 
		PARTITION p1 
	VALUES 
		LESS THAN (2, 1000, 1000), 
		PARTITION p2 
	VALUES 
		LESS THAN (3, 3000, 3000), 
		PARTITION p3 
	VALUES 
		LESS THAN (MAXVALUE, MAXVALUE, MAXVALUE)
);

 CREATE TABLE `company__third_party_clues_base_info_p` (
	`id` int(11) NOT NULL AUTO_INCREMENT, 
	`third_party_clues_id` int(11) NOT NULL, 
	`third_party_id` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`companyName` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`socialSecurityNumber` int(11) NOT NULL DEFAULT '0', 
	`company_id` int(11) NOT NULL DEFAULT '0', 
	`deal_status` int(6) NOT NULL DEFAULT '0', 
	`companySize` varchar(16) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`entType` varchar(32) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`officialweb` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`city` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`city_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`regCapital` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`arInsuranceNumber` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`legalPerson` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`regDate` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`creditNo` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAddr` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAuthority` varchar(80) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry_id` int(10) NOT NULL DEFAULT '0', 
	`openStatus` varchar(64) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`related_company_nums` int(11) NOT NULL DEFAULT '0', 
	`related_contacts_nums` int(11) NOT NULL DEFAULT '0', 
	`related_companys_emplyee_nums` int(11) NOT NULL DEFAULT '0', 
	`state` int(11) NOT NULL, 
	`urlRecords` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	`update_time` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP, 
	`remark` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`del_flag` int(11) NOT NULL, 
	KEY `companyName` (`companyName`), 
	KEY `socialSecrity` (`socialSecurityNumber`), 
	KEY `related_company_nums` (`related_company_nums`), 
	KEY `deal_status` (`deal_status`), 
	KEY `company_id` (
		`related_companys_emplyee_nums`
	) USING BTREE, 
	KEY `id` (`id`) USING BTREE, 
	KEY `thirds_ids` (`third_party_id`) USING BTREE, 
	KEY `third_id` (`third_party_clues_id`) USING BTREE
) ENGINE = MyISAM DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci COMMENT = '第三方线索表基础信息表' 
/*!50100 PARTITION BY KEY (deal_status,companySize,socialSecurityNumber) PARTITIONS 100 */
 

CREATE TABLE `company__third_party_clues_base_info_p2` (
	`id` int(11) NOT NULL AUTO_INCREMENT, 
	`third_party_clues_id` int(11) NOT NULL, 
	`third_party_id` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`companyName` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`socialSecurityNumber` int(11) NOT NULL DEFAULT '0', 
	`company_id` int(11) NOT NULL DEFAULT '0', 
	`deal_status` int(6) NOT NULL DEFAULT '0', 
	`companySize` int(11) NOT NULL DEFAULT '0', 
	`entType` varchar(32) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`officialweb` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`city` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`city_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`regCapital` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`arInsuranceNumber` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`legalPerson` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`regDate` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`creditNo` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAddr` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAuthority` varchar(80) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry_id` int(10) NOT NULL DEFAULT '0', 
	`openStatus` varchar(64) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`related_company_nums` int(11) NOT NULL DEFAULT '0', 
	`related_contacts_nums` int(11) NOT NULL DEFAULT '0', 
	`related_companys_emplyee_nums` int(11) NOT NULL DEFAULT '0', 
	`state` int(11) NOT NULL, 
	`urlRecords` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	`update_time` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP, 
	`remark` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`del_flag` int(11) NOT NULL, 
	KEY `companyName` (`companyName`), 
	KEY `socialSecrity` (`socialSecurityNumber`), 
	KEY `related_company_nums` (`related_company_nums`), 
	KEY `deal_status` (`deal_status`), 
	KEY `company_id` (
		`related_companys_emplyee_nums`
	) USING BTREE, 
	KEY `id` (`id`) USING BTREE, 
	KEY `thirds_ids` (`third_party_id`) USING BTREE, 
	KEY `third_id` (`third_party_clues_id`) USING BTREE
) ENGINE = MyISAM DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci COMMENT = '第三方线索表基础信息表' PARTITION BY RANGE COLUMNS(
	deal_status, socialSecurityNumber, 
	companySize
) 

(
	PARTITION p0 
	VALUES 
		LESS THAN (1, 1, 1), 
		PARTITION p1 
	VALUES 
		LESS THAN (1, 10, 10), 
		PARTITION p2 
	VALUES 
		LESS THAN (1, 50, 50), 
		PARTITION p3 
	VALUES 
		LESS THAN (2, 100, 100), 
		PARTITION p4 
	VALUES 
		LESS THAN (3, 1000, 1000), 
		PARTITION p5 
	VALUES 
		LESS THAN (MAXVALUE, MAXVALUE,MAXVALUE)
) 

mysql> SHOW PROFILES;   ;                                                                                                                                                                  
+----------+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                                                                                                                               |
+----------+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|       17 | 1.87572675 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2    WHERE   companySize > 100                                                            |
|       18 | 2.07466675 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2    WHERE   companySize > 100 OR socialSecurityNumber>100                                |
|       19 | 2.20662050 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info     WHERE   companySize > 100 OR socialSecurityNumber>100                                  |
|       20 | 2.20369700 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info     WHERE   companySize > 200 OR socialSecurityNumber> 200                                 |
|       21 | 1.97166500 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2     WHERE   companySize > 200 OR socialSecurityNumber> 200                              |
|       22 | 2.22661275 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info     WHERE   companySize > 500 OR socialSecurityNumber> 500                                 |
|       23 | 2.06996400 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2     WHERE   companySize > 500 OR socialSecurityNumber> 500                              |
|       24 | 0.00218650 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2     WHERE  ( companySize > 500 OR socialSecurityNumber> 500 ) AND deal_status = 1       |
|       25 | 0.00185900 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info     WHERE  ( companySize > 500 OR socialSecurityNumber> 500 ) AND deal_status = 1          |
|       26 | 2.27813200 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info     WHERE  ( companySize > 500 OR socialSecurityNumber> 500 ) AND deal_status in (1,0)     |
|       27 | 2.00928900 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2      WHERE  ( companySize > 500 OR socialSecurityNumber> 500 ) AND deal_status in (1,0) |
|       28 | 2.07613225 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2      WHERE  ( companySize > 100 OR socialSecurityNumber> 100 ) AND deal_status in (1,0) |
|       29 | 2.33124600 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info      WHERE  ( companySize > 100 OR socialSecurityNumber> 100 ) AND deal_status in (1,0)    |
|       30 | 2.30497525 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info      WHERE  ( companySize > 100 OR socialSecurityNumber> 100 ) AND deal_status in (1,0)    |
|       31 | 2.04205750 | SELECT  count(1), deal_status,companySize,socialSecurityNumber FROM company__third_party_clues_base_info_p2      WHERE  ( companySize > 100 OR socialSecurityNumber> 100 ) AND deal_status in (1,0) |
+----------+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
15 rows in set, 1 warning (0.00 sec)  


SELECT COUNT(1) FROM company__third_party_clues_base_info_p2 PARTITION (p0) ;
SELECT 
	* 
FROM 
	information_schema.partitions 
WHERE 
	TABLE_SCHEMA = 'aibangmang' 
	AND TABLE_NAME = 'company__third_party_clues_base_info_p2' 
	AND PARTITION_NAME IS NOT NULL ;


~~~

## MERGE 
~~~

CREATE TABLE `company__third_party_clues_base_info1` (
	`id` int(11) NOT NULL AUTO_INCREMENT, 
	`third_party_clues_id` int(11) DEFAULT NULL, 
	`third_party_id` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`companyName` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`socialSecurityNumber` int(11) NOT NULL DEFAULT '0', 
	`company_id` int(11) NOT NULL DEFAULT '0', 
	`deal_status` int(6) NOT NULL DEFAULT '0', 
	`companySize` varchar(16) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`entType` varchar(32) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`officialweb` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`city` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`city_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`regCapital` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`arInsuranceNumber` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`legalPerson` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`regDate` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`creditNo` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAddr` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAuthority` varchar(80) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry_id` int(10) NOT NULL DEFAULT '0', 
	`openStatus` varchar(64) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`related_company_nums` int(11) NOT NULL DEFAULT '0', 
	`related_contacts_nums` int(11) NOT NULL DEFAULT '0', 
	`related_companys_emplyee_nums` int(11) NOT NULL DEFAULT '0', 
	`state` int(11) NOT NULL, 
	`urlRecords` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	`update_time` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP, 
	`remark` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`del_flag` int(11) NOT NULL, 
	PRIMARY KEY (`id`), 
	UNIQUE KEY `thirds_ids` (`third_party_id`), 
	KEY `third_party_clues_id` (`third_party_clues_id`), 
	KEY `companyName` (`companyName`), 
	KEY `socialSecrity` (`socialSecurityNumber`), 
	KEY `related_company_nums` (`related_company_nums`), 
	KEY `deal_status` (`deal_status`), 
	KEY `company_id` (
		`related_companys_emplyee_nums`
	) USING BTREE
) ENGINE = MyISAM   DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci COMMENT = '第三方线索表基础信息表'

CREATE TABLE `company__third_party_clues_base_info_all` (
	`id` int(11) NOT NULL AUTO_INCREMENT, 
	`third_party_clues_id` int(11) DEFAULT NULL, 
	`third_party_id` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`companyName` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`socialSecurityNumber` int(11) NOT NULL DEFAULT '0', 
	`company_id` int(11) NOT NULL DEFAULT '0', 
	`deal_status` int(6) NOT NULL DEFAULT '0', 
	`companySize` varchar(16) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`entType` varchar(32) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`officialweb` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`province_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`city` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`city_id` varchar(16) COLLATE utf8_unicode_ci NOT NULL, 
	`regCapital` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`arInsuranceNumber` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`legalPerson` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`regDate` varchar(80) COLLATE utf8_unicode_ci NOT NULL, 
	`creditNo` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAddr` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`regAuthority` varchar(80) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`parent_industry_id` int(10) NOT NULL DEFAULT '0', 
	`openStatus` varchar(64) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`related_company_nums` int(11) NOT NULL DEFAULT '0', 
	`related_contacts_nums` int(11) NOT NULL DEFAULT '0', 
	`related_companys_emplyee_nums` int(11) NOT NULL DEFAULT '0', 
	`state` int(11) NOT NULL, 
	`urlRecords` varchar(200) COLLATE utf8_unicode_ci NOT NULL DEFAULT '', 
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
	`update_time` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP, 
	`remark` varchar(200) COLLATE utf8_unicode_ci NOT NULL, 
	`del_flag` int(11) NOT NULL, 
	PRIMARY KEY (`id`), 
	UNIQUE KEY `thirds_ids` (`third_party_id`), 
	KEY `third_party_clues_id` (`third_party_clues_id`), 
	KEY `companyName` (`companyName`), 
	KEY `socialSecrity` (`socialSecurityNumber`), 
	KEY `related_company_nums` (`related_company_nums`), 
	KEY `deal_status` (`deal_status`), 
	KEY `company_id` (
		`related_companys_emplyee_nums`
	) USING BTREE
)  DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci COMMENT = '第三方线索表基础信息表'

ENGINE = MERGE 
UNION 
	=(company__third_party_clues_base_info11, company__third_party_clues_base_info12,company__third_party_clues_base_info13,company__third_party_clues_base_info14,company__third_party_clues_base_info15) INSERT_METHOD = LAST 

 INSERT  into  company__third_party_clues_base_info14  

SELECT  * FROM  company__third_party_clues_base_info  
WHERE id >=3000000 AND id <4000000  ;


数据合起来了，但是还是占用了一定的空间
SELECT
  TABLE_NAME AS `Table`,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024) AS `Size (MB)`
FROM
  information_schema.TABLES
WHERE
    TABLE_SCHEMA = "aibangmang"
  AND
    TABLE_NAME = "company__third_party_clues_base_info_all"
ORDER BY
  (DATA_LENGTH + INDEX_LENGTH)
DESC  ;

~~~


## INSERT IGNORE 

~~~
INSERT IGNORE INTO `transcripts`
SET `ensembl_transcript_id` = 'ENSORGT00000000001',
`transcript_chrom_start` = 12345,
`transcript_chrom_end` = 12678;


~~~

## SQL_NO_CACHE
~~~
查询时强制不使用缓存
SELECT SQL_NO_CACHE id FROM company

~~~

## mysqldump

~~~
shell :
mysqldump -u root -proot  test user  > c:\temp\my_database.sql ;

 windows ;
 mysqldump.exe  -u root -proot test user > c:\Users\My\Desktop\my_database.sql   


E:\phpstudy2020\MySQL\bin>mysqldump.exe  -u root -proot test user > c:\Users\My\Desktop\my_database.sql  

~~~

## git push  
~~~
 local master overwrite  remote master 
 git  push  -f -u origin master
~~~

## Redis Service
~~~

class RedisService{ 
     protected $redis =  '';
     public function __construct()
     {
         $this->redis = new Redis();
         $this->redis->connect('127.0.0.1',6379);
     }
      
     //缓存字符串 缓存一维数组 二维数组
       
     public function redisCacheStr(){
         $redis = new Redis();
         $redis->connect('127.0.0.1');
         $strCacheKey = 'adolph';
         $arrData = array(
             'name' => 'adolph',
             'age'  => '23',
             'sex'  => '男')

         ;
         $redis->set($strCacheKey,json_encode($arrData));
         $redis->expire($strCacheKey,60);
         $json_data = $redis->get($strCacheKey);
         $data = json_decode($json_data);
         // 这里要删除,防止重复的Key
         $redis->del($strCacheKey);
         var_dump($data);
         $arrWebSite = array(
             'google'=>array(
                 'google.com',
                 'google.com.hk'
             )
         );
         $result = $redis->hSet($strCacheKey,'google',json_encode($arrWebSite['google']));
         $json_data = $redis->hGet($strCacheKey,'google');
         var_dump(json_decode($json_data));
     }
     //计数器
     public function redisCounter(){
         $redis = new Redis();
         $redis->connect('127.0.0.1',6379);
         $counter = 'counts';
         $redis->set($counter,0);
         $redis->incr($counter);
         $redis->incr($counter);
         $redis->incr($counter);
         $strNowCount = $redis->get($counter);
         echo '-------- 当前数量为:' . $strNowCount . ' -------';

     }
     //
      // 获取锁[悲观锁]
      ///@param string $key   关键字
     // @param int $expire   超时时间
      // @return bool
      //
     public function lock($key = '',$expire = 5)
     {
         // 设置值,如果有值就返回失败.  [set if not exists]
         $is_lock = $this->redis->setnx($key,time() + $expire);
         // 有值判断时间.
         if(!$is_lock)
         {
             // 获取有效期.
             $lock_time = $this->redis->get($key);
             // 判断有效期,当前时间如果大于设置的日期  就过期了
             if(time() > $lock_time)
             {
                 // 删除键 并且重新设值.
                 $this->unlock($key);
                 $is_lock = $this->redis->setnx($key,time() + $expire);
             }
         }
         return $is_lock ? true : false;
     }
     public function unlock($key = '')
     {
         return $this->redis->del($key);
     }
      
     //乐观锁
      
     public function optLock(){
         // 乐观锁
         // 解释：乐观锁(Optimistic Lock), 顾名思义，就是很乐观。
         // 每次去拿数据的时候都认为别人不会修改，所以不会上锁。
         // watch命令会监视给定的key，当exec时候如果监视的key从调用watch后发生过变化，则整个事务会失败。
         // 也可以调用watch多次监视多个key。这样就可以对指定的key加乐观锁了。
         // 注意watch的key是对整个连接有效的，事务也一样。
         // 如果连接断开，监视和事务都会被自动清除。
         // 当然了exec，discard，unwatch命令都会清除连接中的所有监视。
         $redis = new Redis();
         $redis->connect('127.0.0.1',6379);
         $strKey = 'pessimistic';
         $redis->set($strKey,10);
         $age = $redis->get($strKey);
         echo '------ Current Age:' . $age . "\n";
         $redis->watch($strKey);
         // 开启事务
         $redis->multi();
         $redis->set($strKey,30);
         $redis->set($strKey,20);
         // 休息30秒让其他去操作
         sleep(30);
         $redis->exec();
         $age = $redis->get($strKey);
         echo "----- Current Age:{$age}\n";

     }
     
     //排行
     
     public  function Top(){
         $redis = new Redis();
         $redis->connect('127.0.0.1',6379);
         $top_str = 'top10';
         // 有序集
         $redis->zAdd($top_str,'50',json_encode(array('name'=>'tom')));
         $redis->zAdd($top_str,'60',json_encode(array('name'=>'adolph')));
         $redis->zAdd($top_str,'40',json_encode(array('name'=>'blank')));
         $dataOne = $redis->zRevRange($top_str,0,-1,true);
         echo '从大到小：';
         print_r($dataOne);
         $dataTwo = $redis->zRange($top_str,0,-1,true);
         echo '从小到大:';
         print_r($dataTwo);
     }
     
    // 队列 
     public function redisQueue(){
         $redis = new Redis();
         $redis->connect('127.0.0.1',6379);
         $strQueueName = 'queue';
         // 进队列.
         $redis->rPush($strQueueName,json_encode(array('uid'=>1,'name'=>'Job')));
         $redis->rPush($strQueueName,json_encode(array('uid'=>2,'name'=>'Tom')));
         $redis->rPush($strQueueName,json_encode(array('uid'=>3,'name'=>'John')));
         echo "------ 进队列成功 ------";
         $strCount = $redis->lRange($strQueueName,0,-1);
         echo '当该队列数据为:' . count($strCount);
         $queue = $redis->lPop($strQueueName);
         echo '获取一个队列';
         $strCount = $redis->lRange($strQueueName,0,-1);
         echo '当前队列为:' . count($strCount);
     }
    
     //订阅客户端[消费端]
       
     public function redisSubscribeClient(){
         ini_set('default_socket_timeout',-1);
         $redis = new Redis();
         $redis->connect('127.0.0.1',6379);
         $strChannel = 'adolph_channel';
         echo '开始订阅' . $strChannel . '频道';
         $redis->subscribe(array($strChannel),function($redis,$channel,$msg){
             print_r(array(
                 'redis'=>$redis,
                 'channel'=>$channel,
                 'msg'=>$msg
             ));
         });
     }
     
     //订阅服务端 
     public function redisSubscribeServer(){
         ini_set('default_socket_timeout',-1);
         $redis  = new Redis();
         $redis->connect('127.0.0.1',6379);
         $strChannel = 'adolph_channel';
         $redis->publish($strChannel,'订阅频道,来自server');
         echo '消息推送成功';
         // 关闭redis
         $redis->close();
     }


}

~~~

## shell for 
~~~
for line in $(cat source.log)
         do
         echo  'https://a.com/Order?dir='$line;
         done 


 for line in $(cat rongorder.log)
         do
         curl https://zqh.huoyanzichan.com/Home/Order/realDealDirOrder?dir=$line;
         done


~~~

## nohup
~~~

 nohup php myprog.php > log.txt &

~~~

## Vi Vim
~~~
    :set enc=utf8
    :set nu
    V 全行选中
    Home  End  
    2yy  2pp  
    G 
    gg  
    gg=G
    200G  跳到200行  
    vim file1 file2 file3
    CTRL-ww  (  上下左右窗口切换)
    :vs  拆成左右
    :sp  拆成上下
    :%s/foo/word/g   在全局范围(%)查找foo并替换为word，所有出现都会被替换（g）。
    :5,12s/foo/word/g  在5-12行查找foo并替换为word，所有出现都会被替换（g）。
    :%s/\\"/ /g
    替换换行与空格
    :%s/\s//g 
    :%s/\r//g 
    :%s/\n//g

    vimDiff 
    ★在两个文件之间来回跳转，可以用下列命令序列：Ctrl-w, w
    ★跳转到下一个diff点：
    ]c
    ★ 跳转到前一个diff点：
    [c
    把一个差异点中当前文件的内容复制到另一个文件里，可以使用命令：
    dp （diff "put"）
    把另一个文件的内容复制到当前行中，可以使用命令：
    do (diff "get"，之所以不用dg，是因为dg已经被另一个命令占用了，所以用了diff "obtain")
~~~

## find 
~~~

    find  ./ -name "*.json"
    find  ./ -name "mysqld.sock"
    find  ./ -name "*.php" -mtime 0  今天修改过的
    find  ./ -name "*.php" -mtime 1  昨天修改过的
    
    file 
    find / -type f -name "*.conf" 
    find / -name 'httpd.conf' -print
    find / -name "*.mp4" -print
    find ./ -type f -exec grep -l "export" {} \;
    find ./ -type f | xargs grep 'export'

    find / -size +10M
    find / -size +10MB 
    find ./ -type f -size +1G
    
    不包含的
    find ./ -iname "*.php" -exec grep -Li "parent::__construct" {} \+

    找到50M以下的 复制过去
    find /home/ubuntu/d1 -type f -size -50000k -exec cp -nv {}  /home/ubuntu/d2   \;

    yes | find /home/ubuntu/d1 -type f -size -50000k -exec cp -nv {}  /home/ubuntu/d2   \;
~~~

## restructArrayIndex rebuild index

~~~

static function restructArrayIndex($array,$column1){
        if(empty($array)){
            return [];
        }

        $returnArr = [];
        foreach ($array as $key=>$value){
            $returnArr[$value[$column1]] =  $value;
        }
        return $returnArr;
}

static function restructArrayIndexV2($array,$column1,$column2){
        if(empty($array)){
            return [];
        }

        $returnArr = [];
        foreach ($array as $key=>$value){
            if(
                $value[$column1]&&
                $value[$column2]
            ){
                $returnArr[$value[$column1]][$value[$column2]] =  $value;
            }
        }

        return $returnArr;
    }

    static function restructArrayIndexV3($array,$column1,$column2,$column3){
        if(empty($array)){
            return [];
        }

        $returnArr = [];
        foreach ($array as $key=>$value){
            if(
                $value[$column1]&&
                $value[$column2]&&
                $value[$column3]
            ){
                $returnArr[$value[$column1]][$value[$column2]][[$value[$column3]]] =  $value;
            }
        }

        return $returnArr;
    }

~~~
 
## code example
~~~  
static function transferMainAgToOrder($agId,$force=1){

    if(!$force){
        if(!\model\Agreements::checkIfIsMainAg($agId)){
            return  \yoka\YsError::error('不是主合同 不允许转为订单'.$agId); ;
        }
    }
        
    $agModel = \model\Agreements::getById($agId);

    
    if(!$force){
        if(self::checkIfMainAgIsOrder($agModel->fxiaoke_id)){
            return  \yoka\YsError::error('已经是订单  不允许转为订单'.$agId); ;
        }    
    }
    
    if($force){
        if(self::checkIfMainAgIsOrder($agModel->fxiaoke_id)){
                
        }    
        else{
            if(!self::replaceAgDataToOrder(" WHERE id = ".$agId)){
                return  \yoka\YsError::error(' 转订单失败'); 
            } 
        }
    }
    else{
        if(!self::replaceAgDataToOrder(" WHERE id = ".$agId)){
            return  \yoka\YsError::error(' 转订单失败'); 
        }
    }
    
    

    $orderModel = self::getByFxiaokeId($agModel->fxiaoke_id); 

    if(!\model\Agreements::setAgsOrderId($agId,$orderModel->id)){
        return  \yoka\YsError::error(' 设定合同的订单id失败'); 
    }

    if(!\model\AgreementsPaylog::setOrderIdByAgId($agId,$orderModel->id)){
        return  \yoka\YsError::error(' 设定回款的订单id失败'); 
    }
        
    if(!\model\agreement\WorkDetail::setOrderIdByAgId($agId,$orderModel->id)){
        return  \yoka\YsError::error(' 设定销售自建工单的订单id失败'); 
    };

    if(!\model\agreement\WorkDetailPayLog::setOrderIdByAgId($agId,$orderModel->id)){
        return  \yoka\YsError::error(' 设定销售自建回款的订单id失败'); 
    } 

    if(!\model\sales\SalesInvoice::setOrderIdByAgId($agId,$orderModel->id)){
        return  \yoka\YsError::error(' 设定发票的订单id失败'); 
    };

    if(!self::copyAgToXiaoShouBao($orderModel,$agId)){
    return  \yoka\YsError::error(' 复制工单失败'); 
    }
        
    // \model\agreement\WorkDetailPlan::$table;
    if(!\model\AgreementsPlan::copyToWorkDetailPlan($agId)){
        return  \yoka\YsError::error(' 复制方案失败'); 
    }; 

    if(!\model\AgreementsPaylog::copyToWorkDetailPaylog($orderModel->id)){

    };

    return  true ;
} 

static function copyToWorkDetailPlan($agId){

        $agModel =  \model\Agreements::getById($agId);
        
        $orderModel =  \model\agreement\Order::getById($agModel->order_id);

        $workDetailModel = \model\agreement\WorkDetail::getByOrderIdAndAgNoAndCompanyId(
            $orderModel->id,
            $orderModel->agreement_no,
            $orderModel->company_id_accredit,
            $orderModel->company_id
        );

        $allPlans = self::getByAgreementId($agId);
        foreach($allPlans as $planItem){
            $detailData = [
                'company_id'=>$planItem['company_id'],
                'work_detail_id'=>$workDetailModel->id,
                'year'=>date('Y',strtotime($planItem['begin_date'])),
                'begin'=>intval(date('m',strtotime($planItem['begin_date']))),
                'end'=>intval(date('m',strtotime($planItem['end_date']))),
                'need_p'=>$planItem['need_num'],
                'need_pm'=>$planItem['need_pm'],
                'days_submit_zl'=>5,
                'days_contract_upload'=>5,
                'days_contract_sign'=>15,
                'days_complete'=>5,
            ];  
           
            if(
                \model\agreement\WorkDetailPlan::getByWorkDetailIdAndDate(
                    $detailData['work_detail_id'],
                    $detailData['year'],
                    $detailData['begin'],
                    $detailData['end']
                )
            ){

            }

            else{
                $newPlanModel =  new \model\agreement\WorkDetailPlan();
                if(
                    !$newPlanModel->add($detailData)
                ){
                   return  \yoka\YsError::error('插入失败'.json_encode($detailData));
                } 
            }

        } 
 
       return true;     
}  
~~~


## grep 
~~~
grep directory 目录下匹配关键字
grep -R 'export'  CodeForCommit/
grep -rnw 'CodeForCommit/' -e 'export'
 i stands for ignore case (optional in your case).
-r or -R is recursive 递归,
-n is line number,  
-w stands for match the whole word. 
l stands for "show the file name, not the result itself".


 包含/不包含关键字的文件
 grep -riL "checkLicense" .
  -L, --files-without-match 不包含匹配的文件
             each file processed.
  -l, --files-with-matches  包含匹配的文件
             Only the names of files containing selected lines are written

 匹配结果前后 匹配新关键词
  grep -R -C 10  "export" a/ | grep 'checkLicense'  

 匹配关键1 不匹配关键字2
 cat a.txt | grep -R   "export" | grep -v 'checkLicense'
-v, --invert-match
          Invert the sense of matching, to select non-matching lines.  (-v
          is specified by POSIX.)

  grep -o 'needle' file | wc -l
  -o will only output the matches, ignoring lines;
  wc can count them:         
~~~
 

 ## du df 
 ~~~
服务器磁盘
df -h 

文件夹大小
du -sh  20200904

多个文件夹大小
du -shc  ./202009* 

du -h ./ | sort -rh | head -5
~~~

## SUM case 
~~~
SELECT  
    SUM(case when kind = 1 then 1 else 0 end) as countKindOne

~~~
 

## mysql table version  
~~~
mysql 数据变更备份

 CREATE TABLE IF NOT EXISTS `table1` (
  `id` int(11) NOT NULL,
   product VARCHAR(100) NOT NULL,
    quantity INT NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS `table1_versions` (
  `id` int(11) NOT NULL,
   product VARCHAR(100) NOT NULL,
    quantity INT NOT NULL DEFAULT 0,
  `idhistory` int(20) NOT NULL auto_increment ,
  `historydate` datetime default NULL ,
  PRIMARY KEY (`idhistory`)
) ;
  
DELIMITER $$ 
CREATE TRIGGER table1_trigger BEFORE UPDATE ON table1 
FOR EACH ROW BEGIN
   INSERT INTO table1_versions 
          SELECT *,null,NOW() FROM table1 WHERE id = OLD.id ; 
END$$ 
DELIMITER ; 

INSERT INTO table1(id,product, quantity)
VALUES
    (1,'2001 Ferrari Enzo',140),
    (2,'1998 Chrysler Plymouth Prowler', 110),
    (3,'1913 Ford Model T Speedster', 120);

mysql> select * from  table1_versions  ;
+----+-------------------+----------+-----------+---------------------+
| id | product           | quantity | idhistory | historydate         |
+----+-------------------+----------+-----------+---------------------+
|  1 | 2001 Ferrari Enzo |      140 |         1 | 2020-09-21 13:55:23 |
|  1 | 2001 Ferrari Enzo |      240 |         2 | 2020-09-21 13:56:26 |
+----+-------------------+----------+-----------+---------------------+
mysql> select * from  table1_versions  ;
+----+--------------------------------+----------+-----------+---------------------+
| id | product                        | quantity | idhistory | historydate         |
+----+--------------------------------+----------+-----------+---------------------+
|  1 | 2001 Ferrari Enzo              |      140 |         1 | 2020-09-21 13:55:23 |
|  1 | 2001 Ferrari Enzo              |      240 |         2 | 2020-09-21 13:56:26 |
|  1 | 240                            |      240 |         3 | 2020-09-21 13:56:44 |
|  2 | 1998 Chrysler Plymouth Prowler |      110 |         4 | 2020-09-21 13:56:44 |
|  3 | 1913 Ford Model T Speedster    |      120 |         5 | 2020-09-21 13:56:44 |
+----+--------------------------------+----------+-----------+---------------------+
5 rows in set (0.00 sec)

~~~





## mysql source  
~~~
  ubuntu@VM-0-4-ubuntu:/home/$ sudo mkdir dbData;
  ubuntu@VM-0-4-ubuntu:/home/$  cd dbData/;
  ubuntu@VM-0-4-ubuntu:/home/dbData$ sudo  rz -y ;
  mysql> source /home/dbData/sales.sql; 
~~~

## mysql update   
~~~
   SELECT id,fxiaoke_id,name,mobile,remark,salary FROM `sales` WHERE name in('李庆','张强');
   根据主键、唯一键 更新 reamrk 字段 
   INSERT INTO sales 
            ( 
                        remark, 
                        salary, 
                        name,
                        id 
            ) 
            VALUES 
            ( 
                        '1', 
                        100, 
                        '李庆',
                        '586' 
            ) 
            , 
            ( 
                        '2', 
                        200, 
                        '张强',
                        '615' 
            ) 
on duplicate KEY UPDATE id=VALUES 
       ( 
              id 
       ) ;
    注意：非空且没有默认值的字段 都需要写上   
     
~~~

## tar 
~~~
压缩 并不减大小
tar -cvf app.tar app/
tar –cvf app.tar *.jpg

压缩 减大小
tar -zcvf app.tar app/

解压
tar -xvf app.tar


下载
sz -bey app.tar


~~~

## mysql update case where    
~~~
    UPDATE users
    SET value = CASE 
        WHEN id in (1,4) THEN 53
        WHEN id = 2 THEN 65
        WHEN id in (3,5) THEN 47
    END
    WHERE id IN (1,2,3,4,5) ;
~~~


## mysql update case where    
~~~
  SELECT   TABLE_NAME AS `Table`,   ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024) AS `Size (MB)` FROM   
  information_schema.TABLES where TABLE_NAME = 'wp_usermeta_new' ;   

~~~


## mysql ARCHIVE  engine 
~~~

mysam engine :
mysql> SELECT   TABLE_NAME AS `Table`,   ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024) AS `Size (MB)` FROM  
    ->  information_schema.TABLES where TABLE_NAME = 'wp_usermeta_new' ;
+-----------------+-----------+
| Table           | Size (MB) |
+-----------------+-----------+
| wp_usermeta_new |       432 |
+-----------------+-----------+

ARCHIVE  engine 
ALTER TABLE wp_usermeta_new DROP  KEY user_id ;
ALTER TABLE wp_usermeta_new AUTO_INCREMENT=0, ENGINE=ARCHIVE;
mysql> SELECT   TABLE_NAME AS `Table`,   ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024) AS `Size (MB)` FROM    information_schema.TABLES where TABLE_NAME = 'wp_usermeta_new';
+-----------------+-----------+
| Table           | Size (MB) |
+-----------------+-----------+
| wp_usermeta_new |        49 |
+-----------------+-----------+
1 row in set (0.01 sec)


  CREATE TABLE `wp_usermeta_new` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) DEFAULT NULL,
  `meta_key` varchar(10) COLLATE utf8_unicode_ci DEFAULT NULL,
  `meta_value` varchar(10) COLLATE utf8_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=ARCHIVE AUTO_INCREMENT=13219999 DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci ;
mysql> desc wp_usermeta_new ;
+------------+-------------+------+-----+---------+----------------+
| Field      | Type        | Null | Key | Default | Extra          |
+------------+-------------+------+-----+---------+----------------+
| id         | int(11)     | NO   | PRI | NULL    | auto_increment |
| user_id    | int(11)     | YES  |     | NULL    |                |
| meta_key   | varchar(10) | YES  |     | NULL    |                |
| meta_value | varchar(10) | YES  |     | NULL    |                |
+------------+-------------+------+-----+---------+----------------+
4 rows in set (0.02 sec)


~~~


## mysql login sh     
~~~
ubuntu@VM-0-4-ubuntu:/var/www/conf$ cat db.conf 
username="root"
password="root"

ubuntu@VM-0-4-ubuntu:/var/www/shell$ cat db_login.sh 

#!/bin/sh
source /var/www/conf/db.conf
# mysql -uroot -proot 
#echo "$username"
mysql  -u"$username" -p"$password"


ubuntu@VM-0-4-ubuntu:/var/www/shell$ bash  db_login.sh
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 515
Server version: 5.7.31-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 


全局
ubuntu@VM-0-4-ubuntu:/var/www/shell$ cat  /home/ubuntu/.bashrc 
PATH=$PATH:/var/www/shell

source ~/.bashrc




~~~



##   watch  every 5 seconds    
~~~
服务器按时执行-6秒 
  watch -n 6   " curl    --request GET  'http://XXXX.org/?_c=crontab&_a=pullAllCompanysListsNewVersion&limit=1000'"
~~~




##   nginx  log formatt
~~~

ubuntu@VM-0-4-ubuntu:/etc/nginx$ cat  nginx.conf 
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
	worker_connections 768;
# multi_accept on;
}

http {

##
# Basic Settings
##

	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
# server_tokens off;

# server_names_hash_bucket_size 64;
# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

##
# SSL Settings
##

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
		ssl_prefer_server_ciphers on;

##
# Logging Settings
	log_format main '$remote_addr - $remote_user [$time_local] '
		'"$request" $status $body_bytes_sent '
		'"$http_referer" "$http_user_agent" '
		'$request_time $upstream_response_time $pipe';
##

	access_log /var/log/nginx/access.log   main;
	error_log /var/log/nginx/error.log;

##
# Gzip Settings
##

	gzip on;
	gzip_disable "msie6";

# gzip_vary on;
# gzip_proxied any;
# gzip_comp_level 6;
# gzip_buffers 16 8k;
# gzip_http_version 1.1;
# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

##
# Virtual Host Configs
##

	include /etc/nginx/conf.d/*.conf;
				   include /etc/nginx/sites-enabled/*;
				   }


#mail {
#	# See sample authentication script at:
#	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
# 
#	# auth_http localhost/auth.php;
#	# pop3_capabilities "TOP" "USER";
#	# imap_capabilities "IMAP4rev1" "UIDPLUS";
# 
#	server {
#		listen     localhost:110;
#		protocol   pop3;
#		proxy      on;
#	}
# 
#	server {
#		listen     localhost:143;
#		protocol   imap;
#		proxy      on;
#	}
#}
ubuntu@VM-0-4-ubuntu:/etc/nginx$ 


sudo nginx -t 

sudo systemctl reload nginx 

ubuntu@VM-0-4-ubuntu:/etc/nginx$ tail -f /var/log/nginx/access.log 
121.69.30.230 - - [28/Sep/2020:16:09:30 +0800] "GET /wordpress/ HTTP/1.1" 200 25349 "-" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36" 0.315 0.028 .
 
~~~


## Nginx access   log 
~~~


http  status  cnt
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
which 404 502
awk '($9 ~ /404/)' /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -rn
awk '($9 ~ /502/)' /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -r
ubuntu@VM-0-4-ubuntu:/etc/nginx$ awk '($9 ~ /404/)' /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -rn
     11 /test.php
     10 /wordpress/wp-admin/admin-ajax.php

who are request
awk -F\" '($2 ~ "/test.php"){print $1}' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -r
 

  awk '{print $4,$10,$13,$26,$27}' access.log
[29/Sep/2020:09:34:07 200 "http://49.232.126.145/wordpress/wp-admin/post.php?post=61&action=edit" 0.356 0.336
[29/Sep/2020:09:34:09 200 "http://49.232.126.145/wordpress/wp-admin/post.php?post=61&action=edit" 0.013 0.013
[29/Sep/2020:09:34:10 200 "http://49.232.126.145/wordpress/wp-admin/post.php?post=61&action=edit" 0.015 0.015
[29/Sep/2020:09:34:11 200 "http://49.232.126.145/wordpress/wp-admin/post.php?post=61&action=edit" 0.028 0.028
[29/Sep/2020:09:34:11 200 "http://49.232.126.145/wordpress/wp-admin/post.php?post=61&action=edit" 0.027 0.027
[29/Sep/2020:09:34:11 200 "http://49.232.126.145/wordpress/wp-admin/post.php?post=61&action=edit" 0.012 0.012

所有200请求
awk '($10 ~ /200/)' access.log  
awk '($10 ~ /200/)' access.log | awk '{print $26}' | sort |head -5 
0.012
0.013
0.013
0.015
0.015
0.017
0.020
0.023
awk '($26 ~ /0.012/)' access.log 

所有非200
awk '($10 !~ /200/)' access.log  

时长大于0.01秒的
cat  access.log | awk '$26 > 0.011' 
cat  access.log | awk '$26 > 0.01 && 0.03 > $6' | wc -l

~~~

 

 ## goaccess  nginx log 
~~~
监控access log  
 sudo apt install goaccess 

 sudo vi  /etc/nginx/nginx.conf
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" '
                '$upstream_response_time $request_time';


sudo vi /etc/goaccess.conf
time-format %T
date-format %d/%b/%Y
log-format %h - %^ [%d:%t %^] "%r" %s %b "%R" "%u"  %^ %T

ubuntu@VM-0-4-ubuntu:/var/log/nginx$ goaccess -f access.log

Dashboard - Overall Analyzed Requests                                                                                                                            [Active Panel: Visitors]

  Total Requests  50 Unique Visitors 0 Unique Files 1 Referrers 0
  Valid Requests  3  Processed Time  0 Static Files 0 Log Size  11.51 KiB
  Failed Requests 47 Excl. IP Hits   0 Unique 404   0 Bandwidth 546.0   B
  Log File        access.log

 > 1 - Unique visitors per day - Including spiders                                                                                                                             Total: 1/1

 Hits Vis.       %   Bandwidth Avg. T.S. Cum. T.S. Max. T.S. Data
 ---- ---- ------- ----------- --------- --------- --------- ----
 3       0 100.00%   546.0   B  43.67 ms 131.00 ms  45.00 ms 29/Sep/2020 ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||

   2 - Top requests (URLs)                                                                                                                                                     Total: 1/1

 Hits Vis.       %   Bandwidth Avg. T.S. Cum. T.S. Max. T.S. Mtd Proto    Data
 ---- ---- ------- ----------- --------- --------- --------- --- -------- ----
 3       0 100.00%   546.0   B  43.67 ms 131.00 ms  45.00 ms --- ---      \x05\x01\x00


  

 生成网页：
 goaccess -f /var/log/nginx/access.log --log-format=COMBINED -a > /var/www/html/vpser.html
 浏览器访问
 http://你的域名/vpser.html#visit_time 


~~~


## nginx access log 
~~~
logrotate nginx log by numbers
sudo systemctl reload nginx :  
 /var/log/nginx/*.log {
                daily
                missingok
                rotate 14
                compress
                delaycompress
                notifempty
                create 0640 www-data adm
                dateext
                dateformat -%Y-%m-%d.log
                sharedscripts
                prerotate
                if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                run-parts /etc/logrotate.d/httpd-prerotate; \
                fi \
                endscript
                postrotate
                invoke-rc.d nginx rotate >/dev/null 2>&1
                endscript
                }

sudo nginx -t
sudo systemctl reload nginx 

~~~


## free 
~~~

ubuntu@VM-0-4-ubuntu:~$ free -h
              total        used        free      shared  buff/cache   available
Mem:           864M        289M         73M         38M        500M        351M
Swap:            0B          0B          0B



~~~


## https page use http   source
~~~
强制使用http的资源
 1:https跳转页面：https://mydomin.com/?_c=work&_a=test
 document.location.href ="http://cdn.aibangmang.org/media/images/logo.png";
 2:
 https展示页
 <html>
<iframe src="https://mydomin.com/?_c=work&_a=test"></iframe>
</html>


~~~


## procedure kill_other_processes
~~~
杀掉mysql连接
SHOW PROCEDURE STATUS;
 show create procedure kill_other_processes;
CALL kill_other_processes();

 DROP PROCEDURE IF EXISTS kill_other_processes ;
  
DELIMITER $$ 
 CREATE PROCEDURE kill_other_processes() 
BEGIN   
  DECLARE finished INT DEFAULT 0; 
  DECLARE proc_id INT; 
  DECLARE proc_id_cursor CURSOR FOR SELECT id
  FROM information_schema.processlist WHERE USER = 'root'; 
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET finished = 1;   
  OPEN proc_id_cursor;
  proc_id_cursor_loop: LOOP
     FETCH proc_id_cursor INTO proc_id; 

    IF finished = 1 THEN 
       LEAVE proc_id_cursor_loop; 
    END IF; 

    IF proc_id <> CONNECTION_ID() THEN  
     KILL proc_id; 
    END IF; 
  END LOOP proc_id_cursor_loop; 
  CLOSE proc_id_cursor;
END$$ 
DELIMITER ; 


~~~


## mysql safe mode 
~~~
插入NULL的时候是否报错 

show VARIABLES LIKE '%sql_mode%';
select @@global.sql_mode ;
mysql  strict_mode. But it allowed not null when insert data .  not  return error 


~~~
 

##    MySQL Scheduled Event
~~~
mysql 事件触发
 
SET GLOBAL event_scheduler = ON;
 
 CREATE TABLE messages (
    id INT PRIMARY KEY AUTO_INCREMENT,
    message VARCHAR(255) NOT NULL,
    created_at DATETIME NOT NULL
); 

REATE EVENT test_event_02
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 MINUTE
ON COMPLETION PRESERVE
DO
   INSERT INTO messages(message,created_at)
   VALUES('Test MySQL Event 2',NOW());


~~~
 

## mysql  self join  index
~~~
 EXPLAIN SELECT 
	log1.id as begin_id, 
	log1.company_id as company_id, 
	log1.old_csm_sales_id as csm_charger, 
	log1.create_time as begin_time, 
	log2.id as end_id, 
	log2.new_csm_sales_id as next_charger, 
	IF(log2.create_time IS  NULL ,'2099-01-01 00:00:00',log2.create_time ) as end_time  
FROM 
	`company__csm_change_log` as log1 
	LEFT JOIN `company__csm_change_log` as log2 ON log1.new_csm_sales_id = log2.old_csm_sales_id 
	AND log1.id < log2.id;

 用不上索引

EXPLAIN SELECT 
	log1.id as begin_id, 
	log1.company_id as company_id, 
	log1.old_csm_sales_id as csm_charger, 
	log1.create_time as begin_time, 
	log2.id as end_id, 
	log2.new_csm_sales_id as next_charger, 
	IF(log2.create_time IS  NULL ,'2099-01-01 00:00:00',log2.create_time ) as end_time  
FROM 
	`company__csm_change_log` as log1 
	LEFT JOIN `company__csm_change_log` as log2 ON log1.new_csm_sales_id = log2.old_csm_sales_id 
	AND log1.id < log2.id
    AND log2.id  > 100 
    WHERE log1.id  > 100  ;

可以用得上索引 
    
~~~

## php export xls  light class 
~~~
快速导出excel 

public function testXlsWriter3($request, $response)
    {  
         
        $writer = new \model\XlsWriter(); 
        $writer->setAuthor('Some Author'); 

        for($i=0;$i<=2;$i++){
            $data = [];
            $rows = \model\ReflushDataIds::getByIdsV2($i*60+1,$i*60+1+60); 
            // $rows = \model\ReflushDataIds::yieldData($rows);  
            foreach ($rows as   $record ){ 
                $data[] = $record; 
            }   
           
            foreach($data as $row){
                $writer->writeSheetRow('Sheet1', $row);
            }
    
        } 

          
        $writer->writeToStdOut();
        $writer->SetOut("example.xlsx");
        //$writer->writeToFile('example.xlsx');
        //echo $writer->writeToString(); 
 
        exit(0);
    }

  

~~~



## php memory
~~~
内存使用
        case 1:
        $model = new \model\Company();
        foreach ($record as $datum){
            

           $companyData= \model\Company::getEntityById(1);
           $company2Data= \model\Company::getEntityById(1);
           $company3Data= \model\Company::getEntityById(1);
             
            $datum['xxx']=$companyData['id'];
            $datum['xxx2']=$company2Data['id'];
            $datum['xxx3']=$company3Data['id'];  

        }

        每loop一次 涨一次   累加


         case 2： select 取的越少   limit 返回的行数越少 占得越少 
         
         可尝试：一次复杂sql  取出必须 
        $model = new \model\Company();
        foreach ($record as $datum){ 

                        $res = $model->fetchAll("select 
                            ag_payLog.*   
                        from 
                            agreements__paylog as ag_payLog 
                            LEFT JOIN agreements__order as ag_ ON ag_payLog.order_id = ag_.id 
                        where 
                        XXXX
                        limit 
                            0, 
                            5");
            $datum['xxx']=$companyData['id'];
            $datum['xxx2']=$company2Data['id'];
            $datum['xxx3']=$company3Data['id'];  

        } 
  

~~~


## yield
~~~

static function testYield($record){
        $model = new \model\company();
        foreach ($record as $datum){
            

            $companyData= \model\Company::getEntityById(1);
            $company2Data= \model\Company::getEntityById(1);
            $company3Data= \model\Company::getEntityById(1);
            $datum['xxx']=$companyData['id'];
            $datum['xxx2']=$company2Data['id'];
            $datum['xxx3']=$company3Data['id']; 
            //  unset($company3Data);
            //  unset($company2Data);
            //  unset($companyData);
            yield $datum;

        }
    }

    public function testXlsWriter2($request, $response)
    {  
        $writer = new \model\XlsWriter(); 
        $writer->setAuthor('Some Author'); 

        for($i=0;$i<=2;$i++){
            $data = [];
            self::echo_memory_usage();
            $rows = \model\ReflushDataIds::getByIdsV2($i*10+1,$i*10+1+10,'id'); 
             
            $rows = self::testYield($rows); 
           // 无论yield 里多少处理   不会产生压力
            foreach ($rows as   $record ){ 
                $data[] = $record; 
            }   
           
            // 只要打开 就会有压力
            foreach($data as $row){
                $writer->writeSheetRow('Sheet1', $row);
            }
    
        } 
            
        var_dump2();
        $writer->writeToStdOut();
        $writer->SetOut("example.xlsx");
        //$writer->writeToFile('example.xlsx');
        //echo $writer->writeToString(); 
 
        exit(0);


    }

~~~


##  strace
~~~
全线追踪
 ps -ef|grep php  

ubuntu   21352 21317 99 11:43 pts/1    00:00:02 /usr/bin/php7.0 while.php
ubuntu   21369 19990  0 11:43 pts/3    00:00:00 grep --color=auto php
root     26325     1  0 Nov20 ?        00:00:16 php-fpm: master process (/etc/php/7.0/fpm/php-fpm.conf)
sudo strace -f  -p  26325   $( -p  26325   | sed 's/\([0-9]*\)/\-p \1/g') 


sudo strace -f $(pidof php-fpm7.0 | sed 's/\([0-9]*\)/\-p \1/g')


~~~



##  php-fpm status
~~~
追踪所有php请求及状态 

  sudo vim /etc/php/7.0/fpm/pool.d/www.conf  
  pm.status_path = /status
  sudo systemctl reload php7.0-fpm
  sudo vim /etc/nginx/sites-enabled/example.com 
   location ~ ^/(status|ping)$ {
                allow 127.0.0.1;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_index index.php;
                include fastcgi_params;
                #fastcgi_pass 127.0.0.1:9000;
                fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
        }

sudo systemctl reload nginx
http://49.232.126.145/status
pool:                 www
process manager:      dynamic
start time:           26/Nov/2020:19:43:58 +0800
start since:          535
accepted conn:        10
listen queue:         0
max listen queue:     0
listen queue len:     0
idle processes:       1
active processes:     1
total processes:      2
max active processes: 1
max children reached: 0
slow requests:        0

http://49.232.126.145/status?full 
************************
pid:                  1712
state:                Running
start time:           26/Nov/2020:19:43:58 +0800
start since:          582
requests:             11
request duration:     106
request method:       GET
request URI:          /status?full
content length:       0
user:                 -
script:               -
last request cpu:     0.00
last request memory:  0

************************
pid:                  1713
state:                Idle
start time:           26/Nov/2020:19:43:58 +0800
start since:          582
requests:             11
request duration:     26725
request method:       GET
request URI:          /wordpress/index.php
content length:       0
user:                 -
script:               /var/www/html/wordpress/index.php
last request cpu:     74.84
last request memory:  4194304

https://www.tecmint.com/enable-monitor-php-fpm-status-in-nginx/
~~~

##  mysql  rank times
~~~
出现次数 排行
注意：此种rank  如果间隔出现  会重新统计   所以用了  把一组数据 放在临时表  或者 根据rank 字段 排序  

SELECT 
	IF (
		@name = follow.name , 
		@rank := @rank + 1, 
		@rank := 1
	) AS cou, 
	@name := follow.name 
FROM 
	(
		SELECT 
			id, 
			name 
		FROM 
			sales__attendance 
		
		LIMIT 
			200
	) as follow, 
	(
		SELECT 
			
			@name := NULL,  
			@rank := 0
	) a ;


    
SELECT 
	attendance1.name, 
	attendance1.date as '今天', 
	attendance2.date as '昨日', 
	attendance1.begin_time as '今天早打卡', 
	attendance1.end_time as '今天晚打卡', 
	attendance2.begin_time as '昨日早打卡', 

	attendance2.end_time as '昨日晚打卡', 
	DATE_FORMAT(
		attendance1.begin_time, "%H-%i-%s"
	), 
	IF(
		DATE_FORMAT(
			attendance1.begin_time, "%H-%i-%s"
		)> '09-30-00', 
		'晚了', 
		'正常'
	) as state ,
	rank_info.cou
FROM 
	sales__attendance as attendance1 
	LEFT JOIN sales__attendance as attendance2 ON attendance1.name = attendance2.name 
	AND DATE_ADD(attendance1.date, INTERVAL -1 DAY) = attendance2.date 
	left JOIN (
		SELECT
		follow.*,
	IF (
		@name = follow.name
		AND @ptype = follow.state,
		@rank :=@rank + 1 ,@rank := 1
	) AS cou,
	@name := follow.name,
	@ptype := follow.state
FROM
	(
		SELECT 
				attendance1.name, 
				attendance1.date  , 
				attendance2.date as '昨日', 
				attendance1.begin_time as '今天早打卡', 
				attendance1.end_time as '今天晚打卡', 
				attendance2.begin_time as '昨日早打卡', 
				attendance2.end_time as '昨日晚打卡', 
				DATE_FORMAT(
					attendance1.begin_time, "%H-%i-%s"
				), 
				'晚了' as  state
				-- IF(
				-- 	DATE_FORMAT(
				-- 		attendance1.begin_time, "%H-%i-%s"
				-- 	)> '09-30-00', 
				-- 	'晚了', 
				-- 	'正常'
				-- ) as state 
			FROM 
				sales__attendance as attendance1 
				LEFT JOIN sales__attendance as attendance2 ON attendance1.name = attendance2.name 
				AND DATE_ADD(attendance1.date, INTERVAL -1 DAY) = attendance2.date 
			WHERE 
				attendance1.name = '田永山' 
				AND DATE_FORMAT(attendance2.date, "%Y-%m") = '2020-11'
				AND  DATE_FORMAT(
						attendance1.begin_time, "%H-%i-%s"
					)> '09-30-00' 
		) follow,
		(
			SELECT
				@rownum := 0,
				@name := NULL ,@ptype := NULL ,@rank := 0
		) a 
	) as  rank_info 
	ON  attendance1.name = rank_info.name 
	AND attendance1.date  = rank_info.date 
WHERE 
	attendance1.name = '田永山' 
	AND DATE_FORMAT(attendance2.date, "%Y-%m") = '2020-11' ;


SELECT
		follow.*,
	IF (
		@name = follow.name
		AND @ptype = follow.state,
		@rank :=@rank + 1 ,@rank := 1
	) AS cou,
	@name := follow.name,
	@ptype := follow.state
FROM
	(
		SELECT 
				attendance1.name, 
				attendance1.date  , 
				attendance2.date as '昨日', 
				attendance1.begin_time as '今天早打卡', 
				attendance1.end_time as '今天晚打卡', 
				attendance2.begin_time as '昨日早打卡', 
				attendance2.end_time as '昨日晚打卡', 
				DATE_FORMAT(
					attendance1.begin_time, "%H-%i-%s"
				), 
				 
				IF(
					DATE_FORMAT(
						attendance1.begin_time, "%H-%i-%s"
					)> '09-30-00', 
					'晚了', 
					'正常'
				) as state 
			FROM 
				sales__attendance as attendance1 
				LEFT JOIN sales__attendance as attendance2 ON attendance1.name = attendance2.name 
				AND DATE_ADD(attendance1.date, INTERVAL -1 DAY) = attendance2.date 
			WHERE 
				attendance1.name = '田永山' 
				AND DATE_FORMAT(attendance2.date, "%Y-%m") = '2020-11'
				ORDER BY  state
		) follow,
		(
			SELECT
				@rownum := 0,
				@name := NULL ,@ptype := NULL ,@rank := 0
		) a  ;    


redis counter ;


SELECT 
	attendance1.name, 
	attendance1.id as id2, 
	attendance1.date, 
	attendance2.date as '昨日', 
	attendance1.begin_time as '今天早打卡', 
	attendance1.end_time as '今天晚打卡', 
	attendance2.begin_time as '昨日早打卡', 
	attendance2.end_time as '昨日晚打卡', 
	DATE_FORMAT(
		attendance1.begin_time, "%H-%i-%s"
	), 
	IF(
		DATE_FORMAT(
			attendance1.begin_time, "%H-%i-%s"
		)> '09-30-00', 
		'晚了', 
		'正常'
	) as state2, 
	(
		select 
			SUM(1) 
		from 
			(
				SELECT 
					IF(
						DATE_FORMAT(
							attendance11.begin_time, "%H-%i-%s"
						)> '09-30-00', 
						'晚了', 
						'正常'
					) as state1, 
					attendance11.id 
				FROM 
					sales__attendance as attendance11 
					LEFT JOIN sales__attendance as attendance2 ON attendance11.name = attendance2.name 
					AND DATE_ADD(
						attendance11.date, INTERVAL -1 DAY
					) = attendance2.date 
				WHERE 
					attendance11.name = '田永山' 
					AND DATE_FORMAT(attendance2.date, "%Y-%m") = '2020-11'
			) as tmp 
		where 
			tmp.state1 = state2 
			AND tmp.id <= id2
	) AS count_by_id 
FROM 
	sales__attendance as attendance1 
	LEFT JOIN sales__attendance as attendance2 ON attendance1.name = attendance2.name 
	AND DATE_ADD(attendance1.date, INTERVAL -1 DAY) = attendance2.date 
WHERE 
	attendance1.name = '田永山' 
	AND DATE_FORMAT(attendance2.date, "%Y-%m") = '2020-11' ;
~~~



## mysql  change log
~~~
CREATE TABLE data (
	  id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
	  timestamp   TIMESTAMP,
	  data1 VARCHAR(255) NOT NULL,
	  data2  DECIMAL(5,2) NOT NULL 
);
 
CREATE TABLE data_log (
	  action VARCHAR(255),
	  action_time   TIMESTAMP NULL DEFAULT NULL,
	  id INT,
	  timestamp   TIMESTAMP NULL DEFAULT NULL,
	  data1 VARCHAR(255) NULL,
	  data2  DECIMAL(5,2) NULL 
);



DROP TRIGGER IF EXISTS ai_data;

DELIMITER $$

CREATE TRIGGER ai_data AFTER INSERT ON data
FOR EACH ROW
BEGIN
  INSERT INTO data_log (action, action_time, id, timestamp, data1, data2)
  VALUES('insert', NOW(), NEW.id, NEW.timestamp, NEW.data1, NEW.data2);
END$$

DELIMITER ;

DROP TRIGGER IF EXISTS au_data;

DELIMITER $$

CREATE TRIGGER au_data AFTER UPDATE ON data
FOR EACH ROW
BEGIN
  INSERT INTO data_log (action, action_time, id, timestamp, data1, data2)
  VALUES('update', NOW(), NEW.id, NEW.timestamp, NEW.data1, NEW.data2);
END$$

DELIMITER ;

DROP TRIGGER IF EXISTS ad_data;

DELIMITER $$

CREATE TRIGGER ad_data AFTER DELETE ON data
FOR EACH ROW
BEGIN
  INSERT INTO data_log (action, action_time, id, timestamp, data1, data2)
  VALUES('delete',NOW(), OLD.id, OLD.timestamp, OLD.data1, OLD.data2);
END$$


DELIMITER ;



INSERT INTO data (timestamp, data1, data2)
VALUES ('2018-10-03 15:23:54', 'some text', 45.28);

UPDATE data
SET data1 = 'updated value'
WHERE data2 = 45.28;

DELETE FROM data
WHERE data1 = 'updated value';


SELECT * FROM data_log;

~~~


##  php  mysql   dbv

~~~
https://dbv.vizuina.com/documentation/#usage-schema-create
git clone https://github.com/victorstanciu/dbv.git

config.php:用户名加密码

/dbv/data/revisions/1
注意：需要是数字 
 update.sql
即可 （注意文件权限）



~~~


## mysql  variable
~~~

-- 定义一个变量 存放学生总数
set  @totalStudents = 0;  

-- 把学生总数查出来 放进变量里
select 
	count(*) into @totalStudents 
from 
	marksheets; 
    
-- 把学生总数查出来	
select 
	@totalStudents, @totalStudents 
from 
	marksheets; 
    
    
SELECT 
	id, 
	score, 
	-- 按照分数排序后 依次排名 
	@curRank := @curRank +1  AS rank ,
	-- 排名对应的分数
	@orderScore := score 
FROM 
	marksheets, 
	-- 初始化变量
	(
		SELECT 
			@curRank := 0, 
			@orderScore := null, 
			@studentNumber := 1, 
			@percentile := 100
	) r 
ORDER BY 
	score DESC ;

--  上述方式漏洞： 两人同一个分数  应该并列第一   

SELECT 
	id, 
	score, 
	-- 如果第二个人的分数 和上次排名的分数一致  那就并列排名  否则就直接按照排序的顺序
	@curRank := IF(@orderScore = score ,@curRank,@orderNumber)   AS rank ,
    -- 排序后的顺序
	@orderNumber := @orderNumber + 1 as orderNumber, 
	-- 排序依据的分数
	@orderScore := score 
FROM 
	marksheets, 
	-- 初始化变量
	(
		SELECT 
			@curRank := 0, 
			@orderScore := null, 
			@orderNumber := 1, 
			@percentile := 100
	) r 
ORDER BY 
	score DESC ;




CREATE TABLE IF NOT EXISTS `marksheets` (
		`id` int(11) NOT NULL, 
		`sales_id` int(11) NOT NULL DEFAULT '0', 
		`name` varchar(20) COLLATE utf8_unicode_ci NOT NULL COMMENT '纷享名称', 
		`date` date NOT NULL, 
		`score` int(11) DEFAULT '0' COMMENT '工作时长单位秒', 
		`state` tinyint(4) NOT NULL DEFAULT '0' COMMENT '0:正常 1:无效', 
		`del_flag` tinyint(4) NOT NULL DEFAULT '0' COMMENT '0:正常 1:无效', 
		`remark` varchar(255) COLLATE utf8_unicode_ci NOT NULL, 
		`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP, 
		`update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
	) ENGINE = MyISAM AUTO_INCREMENT = 3262 DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci  ; 
    
INSERT INTO `marksheets` (
		`id`, `sales_id`, `name`, `date`, `score`, 
		`state`, `del_flag`, `remark`, `create_time`, 
		`update_time`
	) 
VALUES 
	(
		3261, 0, 'A', '2020-10-20', 90, 0, 0, 
		'', '2020-10-23 02:29:43', '2020-12-01 14:58:07'
	), 
	(
		3260, 0, 'A', '2020-10-20', 80, 0, 0, 
		'', '2020-10-23 02:29:39', '2020-12-01 15:16:28'
	), 
	(
		3259, 0, 'B', '2020-10-21', 80, 0, 0, 
		'', '2020-10-23 02:29:35', '2020-12-01 15:16:32'
	), 
	(
		3257, 0, 'B', '2020-10-21', 100, 0, 0, 
		'', '2020-10-23 02:29:26', '2020-12-01 15:16:36'
	), 
	(
		3258, 0, 'C', '2020-10-22', 90, 0, 0, 
		'', '2020-10-23 02:29:30', '2020-12-01 15:16:40'
	), 
	(
		3256, 0, 'C', '2020-10-22', 100, 0, 0, 
		'', '2020-10-23 02:29:22', '2020-12-01 15:16:43'
	);


SELECT 
	name, 
    score,
	@rank := IF(@score = score,@rank,@orderByNum) ,
    @score := score,
    @orderByNum :=@orderByNum+1
FROM 
	(
		SELECT 
			max(score) as score, 
			name 
		FROM 
			marksheets 
		GROUP by 
			name
	) as tmp_score, 
	(
		SELECT 
			@rank := 1,
            @score := 0,
        	@orderByNum := 1
	) as variable 
order by 
	tmp_score.score desc ;
~~~


##  mysql  explode 
~~~ 

db1 存的 company_member_ids：
29,30,4233,140,141,142,143,182,184,183,237,238,239


db2 存的 company_member_ids：
30,4233,140,141,142,143,182,184,183,237,238,239

想比较差异
把字符串按照逗号拆分成多个值，然后入到临时表 比较

按照位置取值：
SELECT 
	SUBSTRING_INDEX(company_member_ids, ',', 6),
    SUBSTRING_INDEX(SUBSTRING_INDEX(company_member_ids,',',1),',',-1) ,
    SUBSTRING_INDEX(SUBSTRING_INDEX(company_member_ids,',',2),',',-1) ,
    SUBSTRING_INDEX(SUBSTRING_INDEX(company_member_ids,',',3),',',-1) ,
	SUBSTRING_INDEX(SUBSTRING_INDEX(company_member_ids,',',4),',',-1)   
FROM 
	service__income_month 
WHERE 
	`type` = '20' 
	AND `month` = '2020-10' 
	AND `admin_id` = '1100' 
LIMIT 
	1;


总共是490个值：
用loop 循环：
DROP PROCEDURE IF EXISTS tysTestExplode;
DELIMITER ;;
CREATE PROCEDURE tysTestExplode()
BEGIN
DECLARE n INT DEFAULT 459;
DECLARE i INT DEFAULT 0;
-- SELECT COUNT(*) FROM table_A INTO n;
 
SET i=0;
WHILE i<n DO 
   REPLACE INTO reflush_data_ids2 (record_id)  
  SELECT  
    SUBSTRING_INDEX(SUBSTRING_INDEX(company_member_ids,',',i),',',-1)  
FROM 
	service__income_month 
WHERE 
	`type` = '20' 
	AND `month` = '2020-10' 
	AND `admin_id` = '1100' 
LIMIT 
	1 ;
  SET i = i + 1;
END WHILE;
End;

结束后：重置符号
DELIMITER ;;
然后调用
CALL tysTestExplode();	
就可以了；




~~~



## mysql group concat 
~~~
 GROUP_CONCAT:
 1:默认长度有限  会导致显示不全
 show variables like 'group_concat_max_len' 
 2：去重（也可排序）
 SELECT 
	GROUP_CONCAT(
		DISTINCT(company_id) SEPARATOR ','
	) 
FROM 
	`agreements__order` 
WHERE 
	company_id = 4844 
GROUP by 
	company_id  
~~~


## mysql group by speed  up 
~~~
mysql  group  by  提速
原始语句：
select 
	a.name, 
	sum(a.count) aSum, 
	avg(a.position) aAVG, 
	b.col1, 
	c.col2, 
	d.col3 
from 
	a 
	join b on (a.bid = b.id) 
	join c on (a.cid = c.id) 
	join d on (a.did = d.id) 
group by 
	a.name, 
	b.id, 
	c.id, 
	d.id ;

We need all of the information from table a for the “group by” 
but we don’t need to execute all the joins before clustering them.
优化后：
select 
	a.name, 
	aSum, 
	aAVG, 
	b.col1, 
	c.col2, 
	d.col3 
from 
	(
		select 
			name, 
			sum(count) aSum, 
			avg(position) aAVG, 
			bid, 
			cid, 
			did 
		from 
			a 
		group by 
			name, 
			bid, 
			cid, 
			did
	) a 
	join b on (a.bid = b.id) 
	join c on (a.cid = c.id) 
	join d on (a.did = d.id) ;




~~~


## mysql user variables
~~~
SELECT  id   INTO @firstId  FROM   admin__user  WHERE  admin_name  in  (

'田永山'

)  LIMIT  1   
;
SELECT   @firstId ;

 

~~~
 
~~~


SELECT 
	id INTO @minId1 
FROM 
	company__third_clues_contact 
WHERE 
	create_time >= '2020-09-01 00:00:00' 
LIMIT 
	1; 
SELECT 
	id INTO @maxId1 
FROM 
	company__third_clues_contact 
WHERE 
	create_time >= '2020-10-01 00:00:00' 
LIMIT 
	1; 
    
SELECT 
	* 
FROM 
	company__third_clues_contact 
WHERE 
	id >= @minId1 
	AND id <= @maxId1 ;


~~~


## ANALYZE TABLE  
~~~
大表 定时执行优化 （索引有时快有时慢等问题 ）
large  table ,  执行优化
ANALYZE TABLE company__third_party_clues_base_info ;

~~~

## QUICK  DELETE 
~~~
快速插入 删除   ：先禁用索引
SET foreign_key_checks = 0;
/* ... your query ... */
SET foreign_key_checks = 1;


ALTER TABLE a DISABLE KEYS
/* ... your query ... */
ALTER TABLE a ENABLE KEYS 


DELIMITER ;;
CREATE PROCEDURE deleRecords()
BEGIN
DECLARE n INT DEFAULT 10;
DECLARE i INT DEFAULT 1;
 
-- SELECT COUNT(*) FROM table_A INTO n;
 
SET i=1;
WHILE i<n DO 
   DELETE QUICK FROM reflush_data_ids WHERE 
   id >=i
   AND id <=(i+5) 
  ;
  SET i = i + 5;
END WHILE;
End;;

SHOW PROCEDURE STATUS ;
CALL deleRecords();
DELIMITER ;


SELECT SLEEP(1);

~~~

##  display  cdn files with login
~~~
登录后才可以展示图片/文件
用需要登录的地址展示静态资源
     public function displayCdnFile($request, $response)
    {

        \AtTools::displayCdnFile(
            $request->domain_name,
            $request->file_path
        );
       
    }

    /** 
	  简易版 通过路径展示文件
	*/
	static function displayCdnFile($domainName,$filePath){
		header("Content-type: application/pdf");
        header("Content-Disposition: inline; filename=0efae8ba05989c3879f77465cd690a98.pdf");
        @readfile($domainName.$filePath);
	}

~~~


## KILL ALL AFTER GREP 
~~~
按关键词 杀掉进程 比如crontab 比如其他
while:关键词  
ps -A | grep crontab | awk '{print $1}' | xargs kill -9 $1
ps aux | grep -ie crontab | awk '{print $2}' | xargs kill -9 

~~~


## locate
~~~
定位文件位置
  locate nginx.conf

~~~

## git config 
~~~

git config --global user.name "your username"

git config --global user.password "your password"



~~~


## mysql logger
~~~
mysql 记录每次更新
create  table  agreements_ad_history like  agreements;   
    DROP TRIGGER IF EXISTS agreements_ad_triggers ;

    DELIMITER $$ 
   CREATE TRIGGER agreements_ad_triggers BEFORE DELETE ON agreements FOR EACH ROW BEGIN INSERT INTO agreements_ad_history 
SELECT 
  `id`, 
  `order_id`, 
  `fxiaoke_id`, 
  `fxiaoke_id_old`, 
  `fxiaoke_sync`, 
  `company_id`, 
  `stage`, 
  `contract_subject`, 
  `company_id_accredit`, 
  `agreement_no`, 
  `money`, 
  `money_total`, 
  `card_money`, 
  `card_remark`, 
  `examin_money`, 
  `examin_remark`, 
  `service_money`, 
  `year_num`, 
  `order_type`, 
  `order_time`, 
  `sign_year`, 
  `sign_date`, 
  `sign_type`, 
  `renewal_type`, 
  `tag`, 
  `agreement_file`, 
  `agreement_tpl`, 
  `pay_type`, 
  `charger`, 
  `charger_mobile`, 
  `charger_mail`, 
  `sales_id`, 
  `department_id`, 
  `department_id_rollin`, 
  `department_id_other`, 
  `marketer_id`, 
  `marketer_id_rollin`, 
  `marketer`, 
  `marketer_mail`, 
  `service_personal`, 
  `ag_contact_json`, 
  `bill_money`, 
  `begin_date`, 
  `member_ss`, 
  `local_province`, 
  `local_city`, 
  `local_area`, 
  `local_province_id`, 
  `local_city_id`, 
  `local_district_id`, 
  `is_goutong`, 
  `is_tomail`, 
  `todo_state`, 
  `todo_date_complete`, 
  `todo_info`, 
  `total_pm`, 
  `need_pm`, 
  `money_pm`, 
  `need_p`, 
  `setted_pm`, 
  `setted_p`, 
  `handle_note`, 
  `report_state`, 
  `employment_sign_contract_personal`, 
  `renew_state`, 
  `renew_note`, 
  `show_renew`, 
  `dispatch_cid`, 
  `first_pay_day`, 
  `apply_position`, 
  `apply_shebao`, 
  `need_funds`, 
  `state`, 
  `create_time`, 
  `update_time`, 
  `del_flag`, 
  `remark` 
FROM 
  agreements 
WHERE 
  id = OLD.id ; END $$

DELIMITER  ;


    create  table  agreements_ai_history like  agreements;
    DROP TRIGGER IF EXISTS agreements_ai_triggers ;

	DELIMITER $$ 
	CREATE TRIGGER agreements_ai_triggers after 
	INSERT 
	ON agreements FOR each row BEGIN 
	INSERT INTO agreements_ai_history 
	SELECT 
		`id`, 
		`order_id`, 
		`fxiaoke_id`, 
		`fxiaoke_id_old`, 
		`fxiaoke_sync`, 
		`company_id`, 
		`stage`, 
		`contract_subject`, 
		`company_id_accredit`, 
		`agreement_no`, 
		`money`, 
		`money_total`, 
		`card_money`, 
		`card_remark`, 
		`examin_money`, 
		`examin_remark`, 
		`service_money`, 
		`year_num`, 
		`order_type`, 
		`order_time`, 
		`sign_year`, 
		`sign_date`, 
		`sign_type`, 
		`renewal_type`, 
		`tag`, 
		`agreement_file`, 
		`agreement_tpl`, 
		`pay_type`, 
		`charger`, 
		`charger_mobile`, 
		`charger_mail`, 
		`sales_id`, 
		`department_id`, 
		`department_id_rollin`, 
		`department_id_other`, 
		`marketer_id`, 
		`marketer_id_rollin`, 
		`marketer`, 
		`marketer_mail`, 
		`service_personal`, 
		`ag_contact_json`, 
		`bill_money`, 
		`begin_date`, 
		`member_ss`, 
		`local_province`, 
		`local_city`, 
		`local_area`, 
		`local_province_id`, 
		`local_city_id`, 
		`local_district_id`, 
		`is_goutong`, 
		`is_tomail`, 
		`todo_state`, 
		`todo_date_complete`, 
		`todo_info`, 
		`total_pm`, 
		`need_pm`, 
		`money_pm`, 
		`need_p`, 
		`setted_pm`, 
		`setted_p`, 
		`handle_note`, 
		`report_state`, 
		`employment_sign_contract_personal`, 
		`renew_state`, 
		`renew_note`, 
		`show_renew`, 
		`dispatch_cid`, 
		`first_pay_day`, 
		`apply_position`, 
		`apply_shebao`, 
		`need_funds`, 
		`state`, 
		`create_time`, 
		`update_time`, 
		`del_flag`, 
		`remark` 
	FROM 
		agreements
	WHERE  id = new.id ; 

	end $$
	
	DELIMITER ; 



    CREATE TABLE `data_tracking` (
  `tracking_id` int(11) NOT NULL AUTO_INCREMENT, 
  `data_id` int(11) NOT NULL, 
  `field` varchar(50) NOT NULL, 
  `old_value` int(11) NOT NULL, 
  `new_value` int(11) NOT NULL, 
  `modified` datetime NOT NULL, 
  `table_name` varchar(80) NOT NULL DEFAULT '', 
  PRIMARY KEY (`tracking_id`), 
  KEY `name_id` (`table_name`, `data_id`)
) ENGINE = MyISAM DEFAULT CHARSET = latin1


	DELIMITER $$

	DROP TRIGGER IF EXISTS  `agreements_au_triggers ` $$

	CREATE TRIGGER `agreements_au_triggers` 
after 
UPDATE 
  ON `agreements` FOR each row BEGIN IF (new.order_id != old.order_id) THEN INSERT INTO data_tracking (
    `data_id`, `field`, `old_value`, `new_value`, 
    `modified`, `table_name`
  ) 
VALUES 
  (
    new.id, "order_id", old.order_id, 
    new.order_id, now(), 'agreements'
  );
end IF;
IF (new.company_id != old.company_id) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "company_id", old.company_id, 
    new.company_id, now(), 'agreements'
  );
END IF;
IF (
  new.company_id_accredit != old.company_id_accredit
) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "company_id_accredit", 
    old.company_id_accredit, new.company_id_accredit, 
    now(), 'agreements'
  );
END IF; 
IF (
  new.contract_subject != old.contract_subject
) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "contract_subject", 
    old.contract_subject, new.contract_subject, 
    now(), 'agreements'
  );
END IF;
IF (
  new.agreement_no != old.agreement_no
) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "agreement_no", old.agreement_no, 
    new.agreement_no, now(), 'agreements'
  );
END IF;
IF (new.money != old.money) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "money", old.money, new.money, 
    now(), 'agreements'
  );
END IF;
IF (new.year_num != old.year_num) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "year_num", old.year_num, 
    new.year_num, now(), 'agreements'
  );
END IF;
IF (new.sign_year != old.sign_year) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "sign_year", old.sign_year, 
    new.sign_year, now(), 'agreements'
  );
END IF;
IF (new.sign_date != old.sign_date) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "sign_date", old.sign_date, 
    new.sign_date, now(), 'agreements'
  );
END IF;
IF (new.sign_date != old.sign_date) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "sign_date", old.sign_date, 
    new.sign_date, now(), 'agreements'
  );
END IF;
IF (
  new.marketer_id != old.marketer_id
) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "marketer_id", old.marketer_id, 
    new.marketer_id, now(), 'agreements'
  );
END IF;
IF (
  new.marketer_id_rollin != old.marketer_id_rollin
) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "marketer_id_rollin", 
    old.marketer_id_rollin, new.marketer_id_rollin, 
    now(), 'agreements'
  );
END IF;
IF (
  new.marketer_id_rollin != old.marketer_id_rollin
) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "marketer_id_rollin", 
    old.marketer_id_rollin, new.marketer_id_rollin, 
    now(), 'agreements'
  );
END IF;
IF (new.need_pm != old.need_pm) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "need_pm", old.need_pm, 
    new.need_pm, now(), 'agreements'
  );
END IF;
IF (new.money_pm != old.money_pm) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "money_pm", old.money_pm, 
    new.money_pm, now(), 'agreements'
  );
END IF;
IF (new.need_p != old.need_p) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "need_p", old.need_p, 
    new.need_p, now(), 'agreements'
  );
END IF;
IF (new.setted_p != old.setted_p) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "setted_p", old.setted_p, 
    new.setted_p, now(), 'agreements'
  );
END IF;
IF (new.setted_p != old.setted_p) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "setted_p", old.setted_p, 
    new.setted_p, now(), 'agreements'
  );
END IF;
IF (
  new.dispatch_cid != old.dispatch_cid
) then INSERT INTO data_tracking (
  `data_id`, `field`, `old_value`, `new_value`, 
  `modified`, `table_name`
) 
VALUES 
  (
    new.id, "dispatch_cid", old.dispatch_cid, 
    new.dispatch_cid, now(), 'agreements'
  );
END IF;
END $$


	DELIMITER ;

    
~~~ 



##  php upload lagre files 
~~~
上传大文件
1: enable php configs  改配置
2: chunk  upload 拆分后上传

example:   plupload  
https://github.com/moxiecode/plupload 

注意配置：
custom.html:
    chunk_size: '2000kb', 设置为chunk上传   
    max_retries: 3,

    {title : "Mp3 files", extensions : "mp3"}   允许的文件类型

upload.php 
$ph = new PluploadHandler(array(
	'target_dir' => 'uploads/', 配置保存文件的路径 注意是和custom.html同级
	'allow_extensions' => 'jpg,jpeg,png,mp3' 允许的文件类型 此处用的mp3测试的  
)); 

~~~



##  php   memery   limit 
~~~ 
内存使用 内存优化
sample1:
            $model = new \model\AgreementsPaylog();  
            $rows = $model->fetchAll("SELECT 
                                    `ap`.* 
                                FROM 
                                    `agreements__paylog` `ap` 
                                    LEFT JOIN agreements__order `a` ON `ap`.order_id = `a`.id 
                                WHERE 
                                    `a`.state = 0 
                                    and `ap`.`state` = '0'
                    "); 
            
            [已用34 MB]
              
                foreach ($rows as $paylogInfo){
                    $tmp[$paylogInfo['order_id']] = $paylogInfo['order_id'];
                    $paylogInfo['order_id'] &&   $ordersPaylogsInfos[$paylogInfo['order_id']][] = $paylogInfo; 
                } 

            [已用35 MB]

                $departmentRs = \model\sales\Department::getSimpleAll();
                  
                $sales = $model_sales->fetchAll();
                   
                $allOrders = $orderModel->fetchAll('state=0');
 
                $allAgs = $orderModel->fetchAll('state=0');
            [已用330 MB]
                foreach ($ordersPaylogsInfos as $order_id => $paylogInfoArr){ 
                        $company = \model\Company::getEntityById($orderInfo['company_id']);
                        
                        $CommissionRs = \model\Commission::getByAgreementId($order_id); 
                         
                        $csmRe = $model_csm->fetchOne(
                            [
                                'state'=>0,
                                'company_id'=>$orderInfo['company_id']
                            ]);
                        
                        $csm_sales = \model\sales\Sales::getEntityById($csmRe->sales_id);
                         
                        foreach ($paylogInfoArr as $index_key => $paylogItem){ 
                             
                            $invoiceRs = \model\sales\SalesInvoice::getInfoByPayId(
                                $paylogItem['id']
                            );

                            $invoiceLogData  = $invoiceLogModel->fetchOne(
                                "invoice_id=".$invoiceRs['id'].
                                " and state=0 and title='财务确认回款' 
                                order by id desc limit 1"
                            );   
                    }
                }
            [已用450+ MB]


sample2:

            $model = new \model\AgreementsPaylog(); 
            [不取任何都的数据 不存任何多余的数据]
            $res = $model->fetchAll("SELECT 
                                    paylog_info.id, 
                                    paylog_info.order_id, 
                                    order_info.order_type, 
                                    order_info.agreement_no, 
                                    order_info.create_time, 
                                    order_info.money_total, 
                                    order_info.year_num, 
                                    paylog_info.pay_account, 
                                    order_info.marketer_id, 
                                    order_info.company_id, 
                                    order_info.company_id, 
                                    ag_info.money, 
                                    order_info.agreement_cost, 
                                    order_info.contract_subject, 
                                    order_info.sign_type, 
                                    'renew', 
                                    order_info.sign_date, 
                                    order_info.need_pm, 
                                    order_info.money, 
                                    paylog_info.department_id, 
                                    paylog_info.department_id_other, 
                                    paylog_info.pay_type, 
                                    paylog_info.pay_event, 
                                    paylog_info.money, 
                                    paylog_info.pay_money, 
                                    paylog_info.expect_date, 
                                    '2021-01-25', 
                                    'diff_day', 
                                    'pay_date', 
                                    'invoice_date', 
                                    paylog_info.pay_tradeno, 
                                    paylog_info.pay_des, 
                                    'invoice_date', 
                                    'money', 
                                    'note', 
                                    'admin_name', 
                                    paylog_info.state, 
                                    paylog_info.des, 
                                    paylog_info.marketer_id_rollin, 
                                    '%', 
                                    'name', 
                                    'rate' 
                                FROM 
                                    `agreements__paylog` paylog_info 
                                    left JOIN agreements__order order_info ON `paylog_info`.order_id = `order_info`.id 
                                    left JOIN agreements ag_info ON `paylog_info`.agreement_id = `ag_info`.id 
                                WHERE 
                                    `paylog_info`.state = 0 
                                    and `paylog_info`.`state` = '0'"); 
            [用32 MB]

~~~ 


## ideas
~~~
    大表排序：按 field1 排序

    维护表：field1 排序表
    id 是排序的顺序 
    record_id 大表的id  

~~~



## lsof -ln  展示当前运行的PHP
~~~

查询正在运行的PHP脚本：
（不用satus?full的话）

lsof -ln |grep php
先找到在用的PHP版本

lsof -ln |grep 'php7.0'
php-fpm7.  1395              0    2w      REG              253,1      425     114978 /var/log/php7.0-fpm.log
php-fpm7.  1395              0    5w      REG              253,1      425     114978 /var/log/php7.0-fpm.log
php-fpm7.  1395              0    8u     unix 0xffff880037194800      0t0      16645 /run/php/php7.0-fpm.sock type=STREAM
php-fpm7.  1403             33    0u     unix 0xffff880037194800      0t0      16645 /run/php/php7.0-fpm.sock type=STREAM
php-fpm7.  9918             33    0u     unix 0xffff880037194800      0t0      16645 /run/php/php7.0-fpm.sock type=STREAM
php-fpm7. 10201             33    0u     unix 0xffff880037194800      0t0      16645 /run/php/php7.0-fpm.sock type=STREAM


~~~


## 监控 DB  
~~~
/**

     http://testadmin.aibangmang.org/?_c=newtest2&_a=autoKill
     http://admin2018.aibangmang.org/?_c=newtest2&_a=autoKill
     CREATE TABLE `db_stock_info` (
	`id` int(11) NOT NULL AUTO_INCREMENT,
	`details` text COLLATE utf8_unicode_ci NOT NULL DEFAULT '',
	`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
	`update_time` timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP,
	PRIMARY KEY (`id`)
    ) ENGINE = MyISAM DEFAULT CHARSET = utf8 COLLATE = utf8_unicode_ci COMMENT = '记录数据库拥堵'
 */
public function autoKill($request, $response)
{

	$model = new \model\Company();
	$allLists = $model->query("SHOW FULL PROCESSLIST", true);

	// 连接数
	$nums = 0;
	// 详细信息
	$infoArr = [];
	foreach ($allLists as $row) {
		$tmp = [
			'语句' => $row['Info'],
			'当前状态' => $row['Command'] . ' ' . $row['State'],
			'用时' => $row['Time'],
			'id' => $row['Id'],
		];
		$infoArr[] = $tmp;
		var_dump($tmp);
		$nums ++ ;

	}

	if( $nums>=20){
		foreach ($allLists as $row) {
			// 超过6秒的干掉
			if ($row['Time']>=5) {
				$res = $model->query("KILL {$row['Id']}", true);

			}

		}

		$model->query("INSERT INTO
			db_stock_info
			( `details`)
			VALUES (
				'".json_encode($infoArr)."'
			)
			");
	}

}


~~~


## 监控 请求 (代码层面)  
~~~
trace table  

入口 生成唯一 id（主键id） 
进入 action 添加到表 （记录pid 方便速查|如果有status?full就不用记录）
action 结束  更新表 
如果时间小于10秒 ，按照id删除记录 
如果时间大于10秒 添加到慢记录表  
 
1：查看当前的进程： 直接查表 
2:查看慢的进程：直接查表
3：监控请求：如果请求过多，报警或者自动杀掉（有pid） 杀掉时候，添加到异常请求表
    

~~~


## mysql 查询不一致  
~~~
代码是root , phpmyadmin是 xx,两者查询的结果不一致：原来表结构坏了，而且前边没坏，只有后边坏了
（查询sql初始用几个in写的 换成join后发现表crash了  而且是只有后边crash了 ）    

~~~

## mysql 简单查询 却很耗时
~~~
    明明简单的语句 就是执行了好几十秒
     select * from `regulater__sign` 
        where 
            1 and 
            user_id in (
                    select 
                        id 
                    from 
                        member 
                    where 
                        real_mobile = '胡海云' or name = '胡海云'
                    ) 
        ORDER BY sign_date DESC,sign_time DESC 
        limit 0, 20 ;

        EXPLAIN ; 
                
        1	SIMPLE	member	NULL	ALL	PRIMARY,real_mobile	NULL	NULL	NULL	11713	19.00	Using where; Using temporary; Using filesort
        1	SIMPLE	regulater__sign	NULL	ref	user_id	user_id	4	aibangmang.member.id	147	100.00	NULL


        explain 之后 也用到索引  扫描行数也少  就是需要二三十秒 
     
        重建索引 OPTIMIZE TABLE `regulater__sign`
        


~~~



##  逻辑代码位置啊 尤其是多端的 要放在logic 
~~~
    多端的 列表 啥的 基本一致  
    如果把逻辑全部放在controller里...  
    需要写三遍 改三遍 麻烦死了
~~~


