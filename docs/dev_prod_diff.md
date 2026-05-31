## Kustomize: отличия dev/prod

| Параметр              | dev                  | prod                     |
|-----------------------|----------------------|--------------------------|
| Реплики               | 1                    | 2 (frontend, bff, user-service, message-service) |
| CPU requests          | 70–150m              | 150–250m                 |
| CPU limits            | 150–300m             | 300–500m                 |
| Memory requests       | 100–192Mi            | 128–256Mi                |
| Memory limits         | 150–256Mi            | 256–512Mi                |
| Теги образов          | latest               | v1.0.0                   |


