<?php

$directory = new RecursiveDirectoryIterator('./database/migrations');
$iterator = new RecursiveIteratorIterator($directory);
$regex = new RegexIterator($iterator, '/^.+\.php$/i', RecursiveRegexIterator::GET_MATCH);

foreach ($regex as $file) {
    $filePath = $file[0];
    $content = file_get_contents($filePath);

    // Проверяем, есть ли в файле миграции создание таблицы
    if (preg_match('/Schema::create\(\s*[\'"]([^\'"]+)[\'"],\s*function\s*\(Blueprint\s*\$table\)\s*\{/', $content)) {
        // Если да, то добавляем проверку на существование таблицы
        $content = preg_replace_callback(
            '/Schema::create\(\s*[\'"]([^\'"]+)[\'"],\s*function\s*\(Blueprint\s*\$table\)\s*\{/',
            function ($matches) {
                $tableName = $matches[1];
                return "if (!Schema::hasTable('{$tableName}')) {\n    Schema::create('{$tableName}', function (Blueprint \$table) {";
            },
            $content
        );
        $content = preg_replace('/\}\);/', "    });\n}", $content);
    }

    // Проверяем, есть ли в файле миграции изменение таблицы
    if (preg_match('/Schema::table\(\s*[\'"]([^\'"]+)[\'"],\s*function\s*\(Blueprint\s*\$table\)\s*\{/', $content)) {
        // Если да, то добавляем проверку на существование столбцов
        $content = preg_replace_callback(
            '/Schema::table\(\s*[\'"]([^\'"]+)[\'"],\s*function\s*\(Blueprint\s*\$table\)\s*\{(.+?)\}\);/s',
            function ($matches) {
                $tableName = $matches[1];
                $tableDefinition = $matches[2];

                // Разбиваем определение таблицы на строки
                $lines = explode("\n", $tableDefinition);
                $newLines = [];

                foreach ($lines as $line) {
                    // Проверяем, содержит ли строка определение столбца
                    if (preg_match('/\$table->(\w+)\(\'(\w+)\'/', $line, $columnMatches)) {
                        $columnType = $columnMatches[1];
                        $columnName = $columnMatches[2];
                        // Добавляем проверку на существование столбца и его добавление
                        $check = "    if (!Schema::hasColumn('{$tableName}', '{$columnName}')) {\n        ";
                        $newLines[] = $check . $line . "\n    }";
                    } else {
                        $newLines[] = $line;
                    }
                }

                // Собираем обратно определение таблицы
                $newTableDefinition = implode("\n", $newLines);
                return "Schema::table('{$tableName}', function (Blueprint \$table) {\n{$newTableDefinition}\n});";
            },
            $content
        );
    }

    // Сохранение изменений в файле
    file_put_contents($filePath, $content);
}

echo "Миграции обновлены с проверками на существование таблиц и столбцов.\n";
