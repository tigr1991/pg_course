# Используем официальный образ Postgres в качестве базы
FROM postgres:latest

# Копируем вашу локальную конфигурацию в контейнер
COPY ./postgresql.conf /etc/postgresql/postgresql.conf

# Обеспечиваем, чтобы Postgres использовал вашу конфигурацию
CMD ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
