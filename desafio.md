1) Identifique as chaves primárias e estrangeiras necessárias para garantir a integridade referencial. Defina-as corretamente.

CREATE TABLE tenant (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    description VARCHAR(255)
);

CREATE TABLE person (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    birth_date DATE,
    metadata JSONB
);

CREATE TABLE institution (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER,
    name VARCHAR(100),
    location VARCHAR(100),
    details JSONB,
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE
);

CREATE TABLE course (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER,
    institution_id INTEGER,
    name VARCHAR(100),
    duration INTEGER,
    details JSONB,
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE,
    FOREIGN KEY (institution_id) REFERENCES institution(id) ON DELETE CASCADE
);

CREATE TABLE enrollment (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER,
    institution_id INTEGER,
    person_id INTEGER,
    enrollment_date DATE,
    status VARCHAR(20),
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE,
    FOREIGN KEY (institution_id) REFERENCES institution(id) ON DELETE SET NULL,
    FOREIGN KEY (person_id) REFERENCES person(id) ON DELETE CASCADE
);

2) Construa índices que consideras essenciais para operações básicas do banco e de consultas possíveis para a estrutura sugerida.

--Tabela person
--  Index para o campo Nome
CREATE INDEX IDX_PERSON_NAME ON person(name);
-- Index para o campo METADATA
CREATE INDEX IDX_PERSON_METADATA ON person USING gin (metadata);

--Tabela institution
--  Index para o campo name
CREATE INDEX IDX_INSTITUTION_NAME ON institution(name);
--  Index para o campo tenant_id 
CREATE INDEX IDX_INSTITUTION_TENANT ON institution (tenant_id);
-- Index para o campo   location
CREATE INDEX IDX_INSTITUTION_LOCATION ON institution (location);
-- Index para o campo detais 
CREATE INDEX IDX_INSTITUTION_DETAILS ON institution USING gin (details);

--Tabela course
-- Index para o campo name 
CREATE INDEX IDX_COURSE_NAME ON course(name);
-- Index para o campo tenant_id
CREATE INDEX IDX_COURSE_TENANT ON course (tenant_id);
-- Index para o campo institution_id
CREATE INDEX IDX_COURSE_INSTITUTION ON course (institution_id);
-- Index para o campo details
CREATE INDEX IDX_COURSE_DETAIL ON course USING gin (details);
-- Index para os campos tenant_id, institution_id
CREATE INDEX IDX_COURSE_TENANT_INSTITUTION ON course(tenant_id, institution_id);

--Tabela enrollment
-- Index para o campo status
CREATE INDEX IDX_ENROLLMENT_STATUS ON enrollment(status);
-- Index para o campo enrollment
CREATE INDEX IDX_ENROLLMENT_TENANT ON enrollment (tenant_id);
-- Index para o campo institution_id
CREATE INDEX IDX_ENROLLMENT_INSTITUTION ON enrollment (institution_id);
-- Index para o campo person_id
CREATE INDEX IDX_ENROLLMENT_PERSON ON enrollment (person_id);
-- Index para o campo enrollment_date
CREATE INDEX IDX_ENROLLMENT_DATE ON enrollment (enrollment_date);
-- Index para os campos tenant_id, institution_id, person_id
CREATE INDEX IDX_ENROLLMENT_TENANT_INSTITUTION_PERSON ON enrollment(tenant_id, institution_id, person_id);
-- Index para os campos tenant_id, status
CREATE INDEX IDX_ENROLLMENT_TENANT_STATUS ON enrollment(tenant_id, status);
-- Index para os campos tenant_id, institution_id, is_active
CREATE INDEX idx_enrollment_tenant_institution ON enrollment (tenant_id, institution_id, is_active);



3) Considere que em enollment só pode existir um único person_id por tenant e institution. Mas institution poderá ser nulo. Como garantir a integridade desta regra?

-- ALTER TABLE enrollment ADD CONSTRAINT UNIQUE_ENROLL_TENANT_INSTITUTION_PERSON UNIQUE (tenant_id, institution_id, person_id);
--A constraint acima não vai garantir a unicidade, pois o valor de institution_id pode ser nulo. O banco vai tratar o valor nulo como não compáravel. 
-- Por este motivo criei duas constraints, buscando garantir a regra citada. 

CREATE UNIQUE INDEX UNIQUE_ENROLL_TENANT_INSTITUTION_PERSON ON enrollment (tenant_id, institution_id, person_id)
WHERE institution_id IS NOT NULL;

CREATE UNIQUE INDEX UNIQUE_ENROLL_TENANT_INSTITUTION_NULL_PERSON ON enrollment (tenant_id, person_id)
WHERE institution_id IS NULL;

4) Caso eu queira incluir conceitos de exclusão lógica na tabela enrollment. Como eu poderia fazer? Quais as alterações necessárias nas definições anteriores?

--Para implementar uma exclusão lógica no ambiente, adicionei duas colunas à tabela enrollment. 
-- A coluna is_active possui valor padrão de true e indica as linhas ativas da tabela. 
-- A coluna deleted_at tem como objetivo salvar informações sobre quando a linha foi removida lógicamente (is_active atualizado para FALSE)

ALTER TABLE enrollment add column is_active BOOLEAN DEFAULT TRUE;
ALTER TABLE enrollment add column deleted_at TIMESTAMP NULL;

/*
Exemplo de update:

UPDATE enrollment
SET 
    is_active = FALSE,
    deleted_at = NOW()  
WHERE 
    tenant_id = 2
    AND institution_id = 1
    AND person_id = 5; 
*/

5) Construa uma consulta que retorne o número de matrículas por curso em uma determinada instituição. Filtre por tenant_id e institution_id obrigatoriamente. Filtre também por uma busca qualquer -full search - no campo metadata da tabela person que contém informações adicionais no formato JSONB. Considere aqui também a exclusão lógica e exiba somente registros válidos.

SELECT 
    c.id AS course_id, c.name AS course_name, COUNT(e.id) AS enrollment_count
FROM 
    enrollment e
JOIN 
    person p ON e.person_id = p.id
JOIN 
    course c ON e.course_id = c.id
WHERE 
    e.tenant_id = :tenant_id -- Necessário informar o valor do tenant_id, subistituindo :tenant_id
    AND e.institution_id = :institution_id -- Necessário informar o valor do institution_id, subistituindo :institution_id
    AND e.is_active = TRUE
    AND (p.metadata::text ILIKE '%' || :string || '%') -- Necessário informar o valor da string, subistituindo :string
GROUP BY 
    c.id, c.name
ORDER BY 
    enrollment_count DESC;

6)Construa uma consulta que retorne os alunos de um curso em uma tenant e institution específicos. Esta é uma consulta para atender a requisição que tem por objetivo alimentar uma listagem de alunos em determinado curso. Tenha em mente que poderá retornar um número grande de registros por se tratar de um curso EAD. Use boas práticas. Considere aqui também a exclusão lógica e exiba somente registros válidos.

--Devido ao volume de dados retornados, pode ser interessante trabalhar com paginação, utilizando as opções LIMIT e OFFSET

SELECT 
    e.id AS enrollment_id,
    p.id AS person_id,
    p.name AS student_name,
    e.enrollment_date,
    e.status
FROM 
    enrollment e
INNER JOIN 
    person p ON e.person_id = p.id
INNER JOIN 
    course c ON e.institution_id = c.institution_id AND e.tenant_id = c.tenant_id
WHERE 
    e.tenant_id = :tenant_id -- substituir :tenant_id pelo valor do tenant_id que será buscado
    AND e.institution_id = :institution_id -- substituir :institution_id pelo valor do institution_id que será buscado
    AND c.id = :course_id -- substituir :course_id pelo valor do course_id que será buscado
    AND e.is_active = true
ORDER BY 
    p.name
LIMIT 50 OFFSET 0;


7) Suponha que decidimos particionar a tabela enrollment. Desenvolva esta ideia. Reescreva a definição da tabela por algum critério que julgues adequado. Faça todos os ajustes necessários e comente-os.

-- Particionamento por data, utilizando a coluna enrollment_date para criar as partições

-- Instrução para criação da tabela
-- para evitar o erro "ERROR:  unique constraint on partitioned table must include all partitioning columns", a primary key da tabela foi definida como id e enrollment_date
CREATE TABLE enrollment (
    id SERIAL,
    tenant_id INTEGER NOT NULL,
    institution_id INTEGER,
    person_id INTEGER NOT NULL,
    enrollment_date DATE NOT NULL,
    status VARCHAR(20),
    is_active BOOLEAN DEFAULT TRUE,
    deleted_at TIMESTAMP NULL,
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE CASCADE,
    FOREIGN KEY (institution_id) REFERENCES institution(id) ON DELETE SET NULL,
    FOREIGN KEY (person_id) REFERENCES person(id) ON DELETE CASCADE,
    PRIMARY KEY (id, enrollment_date)  -- A chave primária inclui a coluna de particionamento
) PARTITION BY RANGE (enrollment_date);

-- criação das partições
CREATE TABLE enrollment_2023 PARTITION OF enrollment
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE enrollment_2024 PARTITION OF enrollment
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');


8) Sinta-se a vontade para sugerir e aplicar qualquer ajuste que achares relevante. Comente-os
