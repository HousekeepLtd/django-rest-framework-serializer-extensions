# AGENTS.md

This file provides guidance to AI coding agents working in this repository.

## What this is

`djangorestframework-serializer-extensions` — a small library (~600 LOC
across 4 modules) of mixins that DRY up Django REST Framework serializers.
It lets serializer fields be chosen *per-view/per-request* (whitelist via
`only`, blacklist via `exclude`, expand related fields via
`expand`/`expand_id_only`), automatically optimizes the underlying queryset
to minimise DB calls, and provides HashId support for exposing external IDs
instead of internal PKs.

This is the **Housekeep fork**
(`HousekeepLtd/django-rest-framework-serializer-extensions`), maintained
because upstream is inactive (last PyPI release 2.0.1, 2020).

## Commands

Develop against the package in editable mode with its test extra:

```bash
pip install -e .[test]   # install test deps from requirements-test.txt
pytest                   # full suite (settings in tests/conftest.py)
tox                      # full Python × Django × DRF matrix
tox -e py313-django52-drf316   # one matrix cell
tox -e black             # formatting check (line-length 79)
tox -e flake8            # lint (package + tests)
tox -e isort             # import-order check

# Run a single test:
pytest tests/test_fields.py::HashIdFieldTests::test_representation_explicit_model_object
```

There is no `manage.py` / installable Django project — tests configure
Django in-memory in `tests/conftest.py:pytest_configure` (sqlite `:memory:`,
settings via `settings.configure`). `ROOT_URLCONF` is `tests.urls`.

Note: `black`/`flake8` target `rest_framework_serializer_extensions` and
`tests` only — **not** `setup.py`. `isort` is the only env that lints
`setup.py`.

## Architecture

The library is four modules under `rest_framework_serializer_extensions/`:

- **`serializers.py`** (the bulk) — composable mixins, combined into
  `SerializerExtensionsMixin` (MRO order matters: `OnlyFieldsMixin`,
  `ExcludeFieldsMixin`, `ExpandableFieldsMixin`, `SerializerHelpersMixin`).
  Ready-made bases `ExtensionsSerializer` and `ExtensionsModelSerializer`
  mix that into DRF's `Serializer`/`ModelSerializer`.
  - `OnlyFieldsMixin` / `ExcludeFieldsMixin` — override `get_fields()` to
    whitelist/blacklist based on `only`/`exclude` sets in the serializer
    **context**. Blacklist/whitelist take precedence over expand.
  - `ExpandableFieldsMixin` — driven by a `Meta.expandable_fields` dict
    (field name → serializer ref or an advanced config dict with
    `serializer`/`many`/`source`/`id_source`/`id_model`/`read_only`). Also
    overrides `get_fields()` and provides `auto_optimize(qs)`, which walks
    the expand instructions and applies
    `select_related`/`prefetch_related`/`Prefetch`. Bounded by
    `MAX_EXPAND_DEPTH`. Serializer references may be import-path strings,
    resolved lazily.
  - `SerializerHelpersMixin` — `hierarchy` property and
    `represent_child()` helper for `SerializerMethodField`s that need to
    render a child serializer with the parent's context (and auto-optimize
    it).

- **`views.py`** — `SerializerExtensionsAPIViewMixin` reads
  `expand`/`expand_id_only`/`only`/`exclude` from query params (when
  `QUERY_PARAMS_ENABLED`, comma-separated or repeated) **or** from
  `extensions_*` view attributes, injects them into the serializer context,
  and auto-optimizes `get_queryset()` when `AUTO_OPTIMIZE` is on.
  `ExternalIdViewMixin` overrides `get_object()` to look up by HashId
  external ID (requires `lookup_field == "pk"`).

- **`fields.py`** — `HashIdField` (de/serialises internal ID ↔ external
  HashId) and hyperlinked variants
  (`HashIdHyperlinkedIdentityField`/`RelatedField`). The model is resolved
  via `GetHashIdModelMixin` from a `model` kwarg, a `get_<field>_model`
  method on the parent, or `parent.Meta.model`.

- **`utils.py`** — settings access
  (`REST_FRAMEWORK["SERIALIZER_EXTENSIONS"]`), the HashId encode/decode
  logic, and `model_from_definition`. **HashIds encode the model's
  ContentType id + the internal id together**, so decoding validates that
  the content type matches the expected model — this is the integrity check
  behind external IDs.

### Key invariants

- The compatibility surface is **DRF, not Django** — the mixins override
  DRF internals (`get_fields`, `run_validation`, `Field.bind`,
  `PrimaryKeyRelatedField`, `SerializerMethodField`, hyperlinked fields,
  `rest_framework.fields.empty`). When bumping DRF, these are what to
  re-verify.
- Configuration is read from
  `settings.REST_FRAMEWORK["SERIALIZER_EXTENSIONS"]` (`HASH_IDS_SOURCE`,
  `AUTO_OPTIMIZE`, `MAX_EXPAND_DEPTH`, `QUERY_PARAMS_ENABLED`), each
  overridable per-serializer via `Meta` or per-view via `extensions_*`
  attributes.

## Version / CI matrix

Supported (see `tox.ini`): Python 3.10–3.13, Django 4.2 / 5.0 / 5.2, DRF
3.14 / 3.15 / 3.16. `__version__` lives in
`rest_framework_serializer_extensions/__init__.py`; bump it and add a dated
CHANGELOG entry before creating a tag.

## Docs

User-facing docs are MkDocs Markdown under `docs/` (`tox -e docs` builds
them). When changing supported versions or public behaviour, update
`docs/overview.md` and `README.md` (both carry a Requirements list) plus
`CHANGELOG.md`.
