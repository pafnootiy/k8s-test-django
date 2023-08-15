FROM nginx/unit:1.25.0-python3.9
ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1

WORKDIR /code
COPY requirements.txt /code/
RUN apt update && apt install -y python3-pip                                  \
    && pip3 install -r requirements.txt                                       \
    && apt remove -y python3-pip                                              \
    && apt autoremove --purge -y                                              \
    && rm -rf /var/lib/apt/lists/* /etc/apt/sources.list.d/*.list
COPY unit_config.json /docker-entrypoint.d/
COPY . /code/
RUN SECRET_KEY=empty python manage.py collectstatic --noinput
EXPOSE 80
VOLUME ["/media"]