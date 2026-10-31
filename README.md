# XA-TRUST / ECM — GSD v1

**Pacote normativo de referência — versão do pacote 1.0.0**

## 1. Finalidade

O Global State Declaration (GSD) é uma declaração estruturada do estado, escopo,
commit de referência, snapshot e dependências de uma versão-alvo.

O GSD é declarativo. Sua validação estrutural e semântica não prova:
- autoria ou autenticidade criptográfica;
- autorização do emissor;
- execução de workflow ou testes;
- imutabilidade do armazenamento;
- integridade do conteúdo remoto;
- proveniência independente;
- conformidade regulatória ou homologação.

## 2. Conteúdo do pacote

- `schemas/gsd-schema-v1.json`: JSON Schema Draft 2020-12.
- `schemas/gsd-receipt-schema-v1.json`: schema separado para receipt de custódia.
- `src/xatrust_gsd/validator.py`: validador L1 + L2.
- `tests/test_gsd_v1.py`: testes de aceitação.
- `examples/gsd-valid-example.json`: vetor sintético válido para teste.
- `docs/VALIDATION-MODEL.md`: modelo de validação e limites.
- `requirements.txt`: dependência de execução.

## 3. Regras normativas principais

### 3.1 Git object format

`git_object_format` identifica o formato do object ID do repositório:
- `sha1`: `commit_reference` com 40 caracteres hexadecimais minúsculos.
- `sha256`: `commit_reference` com 64 caracteres hexadecimais minúsculos.

### 3.2 Snapshot

`snapshot_evidence.digest_algorithm` e `digest` identificam o digest do conteúdo:
- `file` e `manifest` exigem SHA-256;
- `git_tree` exige o mesmo algoritmo declarado em `git_object_format`;
- o comprimento do digest deve corresponder ao algoritmo.

O digest é armazenado sem prefixo. O algoritmo está em campo separado.
A URI localiza o artefato; não é prova de imutabilidade nem de disponibilidade.
O verificador deve recuperar o conteúdo e recalcular o digest.

### 3.3 Fingerprint

`issuer.key_fingerprint` usa a forma `SHA256:<64 hex>`, calculada sobre a
representação DER SubjectPublicKeyInfo (DER-SPKI) da chave pública. O fingerprint
é um identificador da chave, não uma prova de confiança ou autorização.

### 3.4 Dependências

`component` deve ser único dentro do array `dependencies`, independentemente
de `required_version`. O JSON Schema verifica a estrutura; o validador semântico
implementa a unicidade por componente.

### 3.5 URI

O validador de referência permite, por padrão, os esquemas `https`, `s3`, `gs`,
`oci` e `ipfs`. A lista é configurável. Esquemas não autorizados são rejeitados.
A permissão de um esquema não significa que o recurso exista, seja acessível
ou esteja imutavelmente armazenado.

## 4. Camadas de validação

- **L1 — Estrutural:** JSON Schema Draft 2020-12, incluindo formatos.
- **L2 — Semântica:** coerência de algoritmos, URI permitida e unicidade de componentes.
- **L3 — Criptográfica/proveniência:** fora deste pacote; requer implementação
  que recalcule hashes, valide assinatura, issuer, vínculo ao conteúdo e proveniência.
- **L4 — Governança:** fora deste pacote; ECM/OPA avalia autorização, independência
  e política de promoção.

`STRUCTURAL_AND_SEMANTIC_VALID` significa somente aprovação em L1 e L2.

## 5. Estados e resultados

| Situação | Resultado recomendado |
|---|---|
| JSON/schema ou semântica inválida | `REJECT` |
| Hash recalculado difere do declarado | `VERIFICATION_FAILED` |
| Emissor/chave não confiável ou não autorizado | `PROMOTION_DENIED` |
| Evidência obrigatória ausente | `EVIDENCE_LIMITED` |
| L1-L4 aprovadas por verificadores competentes | elegível a `VERIFIED_SCOPE_LIMITED` |

O validador deste pacote só retorna sucesso ou rejeição L1/L2. Não promove claims.

## 6. Receipt de custódia

O receipt é um artefato separado, produzido pelo serviço de custódia, não pelo
emissor. Seu schema está em `gsd-receipt-schema-v1.json`. O receipt deve vincular
o `global_state_id`, o instante de recebimento, a referência de armazenamento,
o digest do objeto efetivamente armazenado e a política de retenção.

A validação do schema do receipt também não prova que o armazenamento é WORM,
append-only ou resistente a administradores privilegiados. Isso exige controles
e evidências próprios.

## 7. Execução local

Requer Python 3.10+.

```bash
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
PYTHONPATH=src python -m xatrust_gsd.validator examples/gsd-valid-example.json
PYTHONPATH=src python -m unittest discover -s tests -v
```

No PowerShell:

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:PYTHONPATH = "src"
python -m xatrust_gsd.validator examples/gsd-valid-example.json
python -m unittest discover -s tests -v
```

## 8. Status deste pacote

Este é um pacote de especificação e referência de implementação preparado nesta
entrega. Nenhum repositório externo foi alterado; nenhum commit, workflow, run_id,
resultado de CI, assinatura, receipt de custódia ou homologação foi produzido.
[30/10, 23:27] Francisco: $schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://xa.finance/schemas/gsd-receipt-v1.json",
  "title": "XA-TRUST GSD Custody Receipt",
  "description": "Sidecar de custódia produzido pelo serviço de recebimento, não pelo emissor do GSD.",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "receipt_version",
    "global_state_id",
    "received_at",
    "storage_reference",
    "stored_object_digest",
    "retention_policy"
  ],
  "properties": {
    "receipt_version": {
      "const": "XA-TRUST-GSD-RECEIPT-v1"
    },
    "global_state_id": {
      "type": "string",
      "pattern": "^GSD-[0-9]{4}-[0-9]{5,}$"
    },
    "received_at": {
      "type": "string",
      "format": "date-time"
    },
    "storage_reference": {
      "type": "string",
      "minLength": 1,
      "maxLength": 1024
    },
    "stored_object_digest": {
      "type": "string",
      "pattern": "^sha256:[0-9a-f]{64}$"
    },
    "retention_policy": {
      "type": "string",
      "minLength": 1,
      "maxLength": 256
    }
  }
}
[30/10, 23:28] Francisco: {
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://xa.finance/schemas/gsd-schema-v1.json",
  "title": "XA-TRUST ECM Global State Declaration",
  "description": "Contrato estrutural GSD v1. Validade estrutural não comprova autenticidade, autorização, imutabilidade ou execução.",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "spec_version",
    "global_state_id",
    "issuer",
    "scope",
    "target_version",
    "git_object_format",
    "commit_reference",
    "snapshot_evidence",
    "dependencies",
    "declared_at"
  ],
  "properties": {
    "spec_version": {
      "const": "XA-TRUST-ECM-v1"
    },
    "global_state_id": {
      "type": "string",
      "pattern": "^GSD-[0-9]{4}-[0-9]{5,}$"
    },
    "issuer": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "identity",
        "key_fingerprint"
      ],
      "properties": {
        "identity": {
          "type": "string",
          "minLength": 1,
          "maxLength": 256
        },
        "key_fingerprint": {
          "type": "string",
          "pattern": "^SHA256:[0-9a-fA-F]{64}$",
          "description": "SHA-256 hexadecimal do DER SubjectPublicKeyInfo (SPKI) da chave pública."
        }
      }
    },
    "scope": {
      "type": "string",
      "minLength": 1,
      "maxLength": 512
    },
    "target_version": {
      "type": "string",
      "minLength": 1,
      "maxLength": 128
    },
    "git_object_format": {
      "enum": [
        "sha1",
        "sha256"
      ]
    },
    "commit_reference": {
      "type": "string"
    },
    "snapshot_evidence": {
      "type": "object",
      "additionalProperties": false,
      "required": [
        "evidence_type",
        "digest_algorithm",
        "digest",
        "artifacts_uri"
      ],
      "properties": {
        "evidence_type": {
          "enum": [
            "file",
            "git_tree",
            "manifest"
          ]
        },
        "digest_algorithm": {
          "enum": [
            "sha1",
            "sha256"
          ]
        },
        "digest": {
          "type": "string",
          "pattern": "^(?:[0-9a-f]{40}|[0-9a-f]{64})$"
        },
        "artifacts_uri": {
          "type": "string",
          "format": "uri",
          "minLength": 1,
          "description": "Localizador do conteúdo; URI, isoladamente, não comprova imutabilidade."
        }
      },
      "allOf": [
        {
          "if": {
            "properties": {
              "digest_algorithm": {
                "const": "sha1"
              }
            },
            "required": [
              "digest_algorithm"
            ]
          },
          "then": {
            "properties": {
              "digest": {
                "pattern": "^[0-9a-f]{40}$"
              }
            }
          }
        },
        {
          "if": {
            "properties": {
              "digest_algorithm": {
                "const": "sha256"
              }
            },
            "required": [
              "digest_algorithm"
            ]
          },
          "then": {
            "properties": {
              "digest": {
                "pattern": "^[0-9a-f]{64}$"
              }
            }
          }
        },
        {
          "if": {
            "properties": {
              "evidence_type": {
                "enum": [
                  "file",
                  "manifest"
                ]
              }
            },
            "required": [
              "evidence_type"
            ]
          },
          "then": {
            "properties": {
              "digest_algorithm": {
                "const": "sha256"
              }
            }
          }
        }
      ]
    },
    "dependencies": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": [
          "component",
          "required_version"
        ],
        "properties": {
          "component": {
            "type": "string",
            "minLength": 1,
            "maxLength": 256
          },
          "required_version": {
            "type": "string",
            "minLength": 1,
            "maxLength": 128
          }
        }
      }
    },
    "declared_at": {
      "type": "string",
      "format": "date-time"
    }
  },
  "allOf": [
    {
      "if": {
        "properties": {
          "git_object_format": {
            "const": "sha1"
          }
        },
        "required": [
          "git_object_format"
        ]
      },
      "then": {
        "properties": {
          "commit_reference": {
            "pattern": "^[0-9a-f]{40}$"
          }
        }
      }
    },
    {
      "if": {
        "properties": {
          "git_object_format": {
            "const": "sha256"
          }
        },
        "required": [
          "git_object_format"
        ]
      },
      "then": {
        "properties": {
          "commit_reference": {
            "pattern": "^[0-9a-f]{64}$"
          }
        }
      }
    }
  ]
}



o$schema": ": 

This repo holds the website for the [Development Containers Specification](https://github.com/devcontainers/spec).

You may view the site at [containers.dev](https://containers.dev).

## Building

If you'd like to build and preview the site yourself, we make it as smooth as possible through a dev container in this repo!

### Dev container

You may build GitHub Pages sites with [Jekyll](https://jekyllrb.com/), which is a Ruby gem. You could manually install these tools on your machine, or you can easily get started with the setup you already need through a dev container!

You may review this repo's dev container in the [`.devcontainer`](https://github.com/devcontainers/containers.dev/tree/gh-pages/.devcontainer) folder.

It is from this [Jekyll Dev Container Template](https://github.com/devcontainers/templates/tree/main/src/jekyll).

### Steps to build and run

* Clone or open this repo in the dev container-supporting editor of your choosing.
     * You may review supporting tools and services [here](https://containers.dev/supporting).
* Reopen this repo in the dev container, so that the container builds and you may develop inside it using the included tools. 
* Once the dev container finishes building, execute the following command in your dev container to start the site: `bundle exec jekyll serve`
* Check out the site! http://localhost:4000/containers.dev/

## Feedback and contributing

If you'd like to provide feedback on or contribute to the dev containers website, please feel free to open an issue or PR in this repo.

For issues on and contributions to the dev container specification itself, please visit the [Dev Containers Spec repo](https://github.com/devcontainers/spec).

## License

License for this repository: https://github.com/devcontainers/containers.dev/blob/gh-pages/LICENSE.
