---
title: Часть 1 - Введение и настройка REPL
date: 2017-08-30
---

Как веб-разработчик, я использую реляционные базы данных каждый день на своей работе, но для меня они словно черный ящик. У меня несколько вопросов:

- В каком формате сохранены данные? (в памяти или на диске)
- Когда они переносятся из памяти на диск?
- Почему у таблицы может быть только один первичный ключ?
- Как откатывается транзакция?
- Какой формат у индексов?
- Когда и как происходит полное считывание таблицы?
- В каком формате сохраняются prepared statement'ы?

Короче, как **работает** база данных?

Чтобы выяснить что к чему, я пишу базу данных с нуля. It's modeled off sqlite, because it is designed to be small with fewer features than MySQL or PostgreSQL, так что я надеюсь иметь больше шансов понять ее. Вся база данных сохранена в одном файле!

# Sqlite

Есть много [документации о внутненностях sqlite](https://www.sqlite.org/arch.html) на их веб-сайте, еще у меня есть копия книги [SQLite Database System: Design and Implementation](https://play.google.com/store/books/details?id=9Z6IQQnX1JEC).

{% include image.html url="assets/images/arch1.gif" description="Архитектура sqlite (https://www.sqlite.org/zipvfs/doc/trunk/www/howitworks.wiki)" %}

Чтобы извлечь или модифицировать данные, запрос проходит через цепочку компонентов. **Фронтенд** состоит из:
- токенизатора
- парсера
- кодо-генератора

На вход во фронтенд подается SQL-запрос. На выходе - байткод для виртуальной машины sqlite (в сущности, компилированная программа, которая выполняет операции с базой данных).

_Бэкенд_ состоит из:
- Виртуальная машина
- B-дерево
- pager
- интерфейс ОС

**Виртуальная машина** принимает байт-код, сгенерированный фронтендом в качестве инструкций. Она может выполнять операции на одной или нескольких таблицах или индексах, которые хранятся в структуре данных, называемой B-деревом. ВМ, в сущности, большая инструкция switch на тип инструкции в байт-коде.

Каждое **B-дерево** состоит из нескольких узлов. Each node is one page in length. B-дерево умеет извлекать страницу с диска или сохранять ее на диск, отправляя команды to the pager.

The **pager**  receives commands to read or write pages of data. It is responsible for reading/writing at appropriate offsets in the database file. It also keeps a cache of recently-accessed pages in memory, and determines when those pages need to be written back to disk.

The **os interface** is the layer that differs depending on which operating system sqlite was compiled for. In this tutorial, I'm not going to support multiple platforms.

[A journey of a thousand miles begins with a single step](https://en.wiktionary.org/wiki/a_journey_of_a_thousand_miles_begins_with_a_single_step), so let's start with something a little more straightforward: the REPL.

## Создание простейшего REPL

Sqlite starts a read-execute-print loop when you start it from the command line:

```shell
~ sqlite3
SQLite version 3.16.0 2016-11-04 19:09:39
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> create table users (id int, username varchar(255), email varchar(255));
sqlite> .tables
users
sqlite> .exit
~
```

To do that, our main function will have an infinite loop that prints the prompt, gets a line of input, then processes that line of input:

```c
int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```

We'll define `InputBuffer` as a small wrapper around the state we need to store to interact with [getline()](http://man7.org/linux/man-pages/man3/getline.3.html). (More on that in a minute)
```c
typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = (InputBuffer*)malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}
```

Затем, `print_prompt()` печатает подсказку пользователю. Делаем это перед каждым чтением пользовательского ввода.

```c
void print_prompt() { printf("db > "); }
```

Чтобы прочитать строчку ввода, испольльзуем [getline()](http://man7.org/linux/man-pages/man3/getline.3.html):
```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```
`lineptr` : a pointer to the variable we use to point to the buffer containing the read line. If it set to `NULL` it is mallocatted by `getline` and should thus be freed by the user, even if the command fails.

`n` : указатель на переменную, в которую сохраняется размер выделенного буфера

`stream` : the input stream to read from. Мы будем считывать стандартный поток ввода.

`return value` : количество считанных байт, которых может быть меньше, чем размер всего буффера.

Мы говорим `getline` сохранить считанную строку в `input_buffer->buffer`, а размер выделенного буфера в `input_buffer->buffer_length`. Возвращенное значение сохраняем в `input_buffer->input_length`.

`buffer` сначала null, таким образом `getline` выделяет достаточно памяти для считанной строчки и makes `buffer` point to it.

```c
void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}
```

Now it is proper to define a function that frees the memory allocated for an
instance of `InputBuffer *` and the `buffer` element of the respective
structure (`getline` allocates memory for `input_buffer->buffer` in
`read_input`).

```c
void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}
```

Finally, we parse and execute the command. There is only one recognized command right now : `.exit`, which terminates the program. Otherwise we print an error message and continue the loop.

```c
if (strcmp(input_buffer->buffer, ".exit") == 0) {
  close_input_buffer(input_buffer);
  exit(EXIT_SUCCESS);
} else {
  printf("Unrecognized command '%s'.\n", input_buffer->buffer);
}
```

Давайте попробуем!
```shell
~ ./db
db > .tables
Unrecognized command '.tables'.
db > .exit
~
```

Ну вот, у нас есть рабочий REPL. В следующей части , we'll start developing our command language. Тем временем, вот вся программа из этой части:

```c
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}

void print_prompt() { printf("db > "); }

void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Игнорировать перевод строки в конце
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}

void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}

int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```
