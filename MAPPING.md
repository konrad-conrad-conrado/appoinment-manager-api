# Mapeo Java (Spring) -> FastAPI modular

Capa técnica Java -> archivo en cada módulo:

| Java            | FastAPI          |
|-----------------|------------------|
| controller/     | router.py        |
| dto/            | schemas.py       |
| entity/         | models.py        |
| repository/     | repository.py    |
| service/        | service.py       |
| exception/      | exceptions.py (del módulo) o core/exceptions.py (compartidas) |
| security/       | core/security.py + core/dependencies.py |
| config/         | core/config.py y main.py |

Regla modular: un módulo solo importa de otro lo que su __init__.py expone
(servicios), nunca models.py ni repository.py de otro módulo.

Ojo: ChatbotService.java usa ServicioRepository directo. En el modular debe
pasar por servicios.service.

Orden sugerido: core -> auth -> servicios -> clientes -> citas -> integraciones -> chatbot.
