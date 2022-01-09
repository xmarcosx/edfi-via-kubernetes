# Python image to use.
FROM python:3.8

# Set the working directory to /app
WORKDIR /app

RUN pip install Flask gunicorn

COPY . .

# Run app.py when the container launches
ENTRYPOINT ["python", "app.py"]
