# canal windows下部署

__资源__  
https://github.com/alibaba/canal?tab=readme-ov-file  
https://github.com/xingwenge/canal-php

__步骤__  
1.需要打开 mysql binlog 写入功能，并配置为`binlog-format`为`ROW`模式  
<img width="507" height="164" alt="image" src="https://github.com/user-attachments/assets/c5ea2ec9-e843-4978-9b16-8a967fa4c738" />  
2.新增 mysql 用户管理员账密为`canal`  
```
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```
3.修改配置`conf/example/instance.properties`，与服务器保持一致  
4.引入`xingwenge/canal-php`，建立监听  
ListenDatabaseCommand.php  
```
<?php

namespace app\common\command;

use app\common\model\Log as LogModel;
use Exception;
use extend\canal\CanalDispose;
use think\console\Command;
use think\console\Input;
use think\console\Output;
use think\facade\Env;
use xingwenge\canal_php\CanalConnectorFactory;

/**
 * 监听数据库变化
 * 
 * @author ltj
 */
class ListenDatabaseCommand extends Command
{
    protected function configure()
    {
        $this->ignoreValidationErrors();
        $this->setName('listen_database');
    }

    protected function execute(Input $input, Output $output)
    {
        # 通过 subscribe 控制要监听的库、表
        $subscribe_tab = [
            Env::get('database.database') . '\\.' . Env::get('database.prefix') . 'material_data_search' => [
                'table' => Env::get('database.prefix') . 'material_data_search',
                'call'  => new \app\common\controller\sync_elasticsearch\MaterialDataSearchController(),
            ]
        ];

        try {
            $client = CanalConnectorFactory::createClient(CanalConnectorFactory::CLIENT_SOCKET);
            $client->connect(Env::get('canal.host'), Env::get('canal.port'));
            $client->checkValid();
            $client->subscribe(Env::get('canal.client_id'), Env::get('canal.destination'), implode(',', array_keys($subscribe_tab)));

            while (true) {
                $message = $client->get(100);

                if ($entries = $message->getEntries()) {
                    foreach ($entries as $entry) {
                        $data = CanalDispose::listen($entry);

                        if (!empty($data)) {
                            foreach ($subscribe_tab as $item) {
                                if ($item['table'] == $data['table']) {
                                    call_user_func_array([$item['call'], underline_to_littlecamelcase($data['type'])], [$data['data']]);
                                    break;
                                }
                            }
                        }
                    }
                }

                sleep(1);
            }

            $client->disConnect();
        } catch (Exception $e) {
            # 日志写入
        }

        return true;
    }
}
```

CanalDispose.php
```
<?php

namespace extend\canal;

use Com\Alibaba\Otter\Canal\Protocol\Column;
use Com\Alibaba\Otter\Canal\Protocol\Entry;
use Com\Alibaba\Otter\Canal\Protocol\EntryType;
use Com\Alibaba\Otter\Canal\Protocol\EventType;
use Com\Alibaba\Otter\Canal\Protocol\RowChange;
use Com\Alibaba\Otter\Canal\Protocol\RowData;

class CanalDispose
{
    /**
     * @param Entry $entry
     * @throws \Exception
     */
    public static function println($entry)
    {
        switch ($entry->getEntryType()) {
            case EntryType::TRANSACTIONBEGIN:
            case EntryType::TRANSACTIONEND:
                return;
                break;
        }

        $rowChange = new RowChange();
        $rowChange->mergeFromString($entry->getStoreValue());
        $evenType = $rowChange->getEventType();
        $header = $entry->getHeader();

        echo sprintf("================> binlog[%s : %d], name[%s,%s], eventType: %s", $header->getLogfileName(), $header->getLogfileOffset(), $header->getSchemaName(), $header->getTableName(), $header->getEventType()), PHP_EOL;
        echo $rowChange->getSql(), PHP_EOL;

        /** @var RowData $rowData */
        foreach ($rowChange->getRowDatas() as $rowData) {
            switch ($evenType) {
                case EventType::DELETE:
                    self::ptColumn($rowData->getBeforeColumns());
                    break;
                case EventType::INSERT:
                    self::ptColumn($rowData->getAfterColumns());
                    break;
                default:
                    echo '-------> before', PHP_EOL;
                    self::ptColumn($rowData->getBeforeColumns());
                    echo '-------> after', PHP_EOL;
                    self::ptColumn($rowData->getAfterColumns());
                    break;
            }
        }
    }

    public static function getList($entry)
    {
        switch ($entry->getEntryType()) {
            case EntryType::TRANSACTIONBEGIN:
            case EntryType::TRANSACTIONEND:
                return [];
        }

        $rowChange = new RowChange();
        $rowChange->mergeFromString($entry->getStoreValue());
        $evenType = $rowChange->getEventType();
        $header = $entry->getHeader();
        $mysqlType = '';
        /** @var RowData $rowData */
        $data = [];

        foreach ($rowChange->getRowDatas() as $rowData) {
            switch ($evenType) {
                case EventType::DELETE:
                    $mysqlType = 'delete';
                    $data[] = self::getColumn($rowData->getBeforeColumns());
                    break;
                case EventType::INSERT:
                    $data[] = self::getColumn($rowData->getAfterColumns());
                    $mysqlType = 'insert';
                    break;
                default:
                    $mysqlType = 'update';
                    $data[] = self::getColumn($rowData->getAfterColumns());
                    break;
            }
        }

        return [
            'table' => $header->getTableName(),
            'data'  => $data,
            'type'  => $mysqlType
        ];
    }

    private static function getColumn($columns, $onlyChange = false)
    {
        /** @var Column $column */
        $data = [];

        foreach ($columns as $column) {
            if ($onlyChange) {
                if ($column->getUpdated()) {
                    $data[$column->getName()] = $column->getValue();
                }
            } else {
                $data[$column->getName()] = $column->getValue();
            }
        }

        return $data;
    }

    private static function ptColumn($columns)
    {
        /** @var Column $column */
        foreach ($columns as $column) {
            echo sprintf("%s : %s  update= %s", $column->getName(), $column->getValue(), var_export($column->getUpdated(), true)), PHP_EOL;
        }
    }

    /**
     * @param Entry $entry
     * @return array|false
     * @throws \Exception
     */
    public static function listen($entry)
    {
        switch ($entry->getEntryType()) {
            case EntryType::TRANSACTIONBEGIN:
            case EntryType::TRANSACTIONEND:
                return [];
        }

        $rowChange = new RowChange();
        $rowChange->mergeFromString($entry->getStoreValue());
        $evenType = $rowChange->getEventType();
        $header = $entry->getHeader();
        $mysqlType = '';
        /** @var RowData $rowData */
        $data = [];

        if (count($rowChange->getRowDatas()) > 2000) {
            return false;
        }

        foreach ($rowChange->getRowDatas() as $rowData) {
            switch ($evenType) {
                case EventType::DELETE:
                    $mysqlType = 'delete';
                    $data[] = self::getColumn($rowData->getBeforeColumns());
                    break;
                case EventType::INSERT:
                    $data[] = self::getColumn($rowData->getAfterColumns());
                    $mysqlType = 'insert';
                    break;
                default:
                    $mysqlType = 'update';
                    $after = self::getColumn($rowData->getAfterColumns(), true);
                    $data[] = [
                        'before' => self::getColumn($rowData->getBeforeColumns()),
                        'after'  => $after,
                    ];
                    break;
            }
        }

        return [
            'table' => $header->getTableName(),
            'data'  => $data,
            'type'  => $mysqlType
        ];
    }
}

```
