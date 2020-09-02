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