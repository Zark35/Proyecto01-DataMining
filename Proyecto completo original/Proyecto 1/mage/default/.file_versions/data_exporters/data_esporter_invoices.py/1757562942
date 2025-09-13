from mage_ai.data_preparation.decorators import data_exporter
from mage_ai.data_preparation.shared.secrets import get_secret_value
from sqlalchemy import create_engine, Table, Column, String, Integer, DateTime, JSON, MetaData
from sqlalchemy.dialects.postgresql import insert


def get_table(table_name: str):
    """
    Define la tabla qb_invoices con esquema estándar raw
    """
    metadata = MetaData()
    return Table(
        table_name, metadata,
        Column('id', String, primary_key=True),
        Column('payload', JSON),
        Column('ingested_at_utc', DateTime),
        Column('extract_window_start_utc', String),
        Column('extract_window_end_utc', String),
        Column('page_number', Integer),
        Column('page_size', Integer),
        Column('request_payload', String),
        extend_existing=True
    )


@data_exporter
def export_to_postgres(data, *args, **kwargs):
    """
    Data Exporter:
    - Inserta los registros en Postgres
    - UPSERT por id (idempotencia)
    """
    user = get_secret_value('pg_user')
    password = get_secret_value('pg_password')
    host = get_secret_value('pg_host')
    port = get_secret_value('pg_port')
    db = get_secret_value('pg_db')

    url_conn = f'postgresql://{user}:{password}@{host}:{port}/{db}'
    engine = create_engine(url_conn)

    table = get_table('qb_invoices')
    with engine.begin() as conn:
        for rec in data:
            stmt = insert(table).values(**rec)
            stmt = stmt.on_conflict_do_update(
                index_elements=['id'],
                set_={k: stmt.excluded[k] for k in rec.keys() if k != 'id'}
            )
            conn.execute(stmt)

    print(f"✅ {len(data)} registros exportados con UPSERT en qb_invoices")
