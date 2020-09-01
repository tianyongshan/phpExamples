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
~~~

