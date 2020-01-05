---
title: Часть 2 - Простейший компилятор SQL и виртуальная машина
date: 2017-08-31
---

Мы создаем клон sqlite. "Фронтенд" для sqlite - это компилятор SQL, который разбирает строку и выдает внутреннее представление, которое называется байт-код.

Этот байт-код передается на исполнение виртуальной машине.

{% include image.html url="assets/images/arch2.gif" description="SQLite Architecture (https://www.sqlite.org/arch.html)" %}

Разделение процесса на два шага имеет пару преимуществ:
- Уменьшение сложности каждого из шагов (например, виртуальная машину не заботят синтаксические ошибки)
- Можно позволить себе компилировать запросы один раз и сохранять байт-код в кэше для улучшения производительности

Принимая это во внимание, давайте отрефакторим нашу функцию `main` и добавим поддержку двух новых ключевых слов:

```diff
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);

-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```

Не-SQL инструкции, вроде `.exit`, называеются "мета-командами". Все они начинаются с точки, таким образом, мы проверяем их и обрабатываем в отдельной функции.

Потом, мы добавляем шаг, на котором мы преобразуем строчку ввода во внутреннее представление инструкции. Это наша наколеночная версия фронтенда sqlite.

Наконец, мы передаем нашу подготовленную инструкцию в `execute_statement`. Эта функция в конечном итоге станет нашей виртуальной машиной.

Обратите внимание, что две из наших фукнций возвращают перечисления, которые указывают на успех или неудачу:

```c
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;

typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
```

"Unrecognized statement"? Это немного похоже на исключение. Но [исключения - это плохо](https://www.youtube.com/watch?v=EVhCUSgNbzo) (и C даже не поддерживает их), поэтому я использую коды ошибок в перечислениях всезде, где это кажется практичным. Компилятор языка C will complain if my switch statement doesn't handle a member of the enum, so we can feel a little more confident we handle every result of a function. Ожидается, что в будущем появятся новые коды.

`do_meta_command` просто обертка над существующей функциональностью, которая оставляет нам место для добавления еще большего числа команд:

```c
MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
  if (strcmp(input_buffer->buffer, ".exit") == 0) {
    exit(EXIT_SUCCESS);
  } else {
    return META_COMMAND_UNRECOGNIZED_COMMAND;
  }
}
```

Наши "prepared statement" прямо сейчас просто содержат перечисление с двумя возможными значениями. Они будут содержать больше данных, как только добавим параметры в запросах:

```c
typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;

typedef struct {
  StatementType type;
} Statement;
```

`prepare_statement` (наш "SQL компилятор") прямо сейчас не понимает ситнаксис SQL. По факту, он только понимает два слова:
```c
PrepareResult prepare_statement(InputBuffer* input_buffer,
                                Statement* statement) {
  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
    statement->type = STATEMENT_INSERT;
    return PREPARE_SUCCESS;
  }
  if (strcmp(input_buffer->buffer, "select") == 0) {
    statement->type = STATEMENT_SELECT;
    return PREPARE_SUCCESS;
  }

  return PREPARE_UNRECOGNIZED_STATEMENT;
}
```

Отметим использование `strncmp` для "insert", ведь за ключевым словом "insert" будут следовать данные. (например `insert 1 cstack foo@bar.com`)

И вот, ставим в `execute_statement` пару заглушек:
```c
void execute_statement(Statement* statement) {
  switch (statement->type) {
    case (STATEMENT_INSERT):
      printf("This is where we would do an insert.\n");
      break;
    case (STATEMENT_SELECT):
      printf("This is where we would do a select.\n");
      break;
  }
}
```

Note that it doesn't return any error codes because there's nothing that could go wrong yet.
Примечательно, что функция пока ничего не возвращает, потому что здесь пока нечему пойти не так.

С проведенным рефакторингом теперь мы понимаем два новых ключевых слова!
```command-line
~ ./db
db > insert foo bar
This is where we would do an insert.
Executed.
db > delete foo
Unrecognized keyword at start of 'delete foo'.
db > select
This is where we would do a select.
Executed.
db > .tables
Unrecognized command '.tables'
db > .exit
~
```

Каркас нашей базы данных начинает принимать форму... неплохо было бы хранить в ней данные? В следующей части мы реализуем `insert` и `select`, создав худшее в мире хранилище данных. Тем временем, вот полный дифф из этой части:

```diff
@@ -10,6 +10,23 @@ struct InputBuffer_t {
 } InputBuffer;
 
+typedef enum {
+  META_COMMAND_SUCCESS,
+  META_COMMAND_UNRECOGNIZED_COMMAND
+} MetaCommandResult;
+
+typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
+
+typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;
+
+typedef struct {
+  StatementType type;
+} Statement;
+
 InputBuffer* new_input_buffer() {
   InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
   input_buffer->buffer = NULL;
@@ -40,17 +57,67 @@ void close_input_buffer(InputBuffer* input_buffer) {
     free(input_buffer);
 }
 
+MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
+  if (strcmp(input_buffer->buffer, ".exit") == 0) {
+    close_input_buffer(input_buffer);
+    exit(EXIT_SUCCESS);
+  } else {
+    return META_COMMAND_UNRECOGNIZED_COMMAND;
+  }
+}
+
+PrepareResult prepare_statement(InputBuffer* input_buffer,
+                                Statement* statement) {
+  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
+    statement->type = STATEMENT_INSERT;
+    return PREPARE_SUCCESS;
+  }
+  if (strcmp(input_buffer->buffer, "select") == 0) {
+    statement->type = STATEMENT_SELECT;
+    return PREPARE_SUCCESS;
+  }
+
+  return PREPARE_UNRECOGNIZED_STATEMENT;
+}
+
+void execute_statement(Statement* statement) {
+  switch (statement->type) {
+    case (STATEMENT_INSERT):
+      printf("This is where we would do an insert.\n");
+      break;
+    case (STATEMENT_SELECT):
+      printf("This is where we would do a select.\n");
+      break;
+  }
+}
+
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);
 
-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      close_input_buffer(input_buffer);
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```
