# Admin MH Credits Grant Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Админ в карточке пользователя видит остаток Мурликов, начисляет +N к лимиту текущего billing-периода и может отозвать начисление.

**Architecture:** Отдельная таблица `billing_mh_credits_admin_grants`. Сумма активных грантов (тот же `period_start`, что у текущего периода пользователя, `revoked_at IS NULL`) входит в `BillingService._apply_limit_boosts` рядом с промо и докупками. Admin API под `require_superuser`; UI — блок на `users/[id]`.

**Tech Stack:** FastAPI / SQLAlchemy / Alembic / Pydantic; admin-panel Next.js / React / TypeScript.

**Spec:** `docs/superpowers/specs/2026-11-09-admin-mh-credits-grant-design.md`

## Global Constraints

- Начисление = +N к `max_mh_credits_per_period`, ledger `used` не трогаем.
- Срок = текущий billing-период; активность гранта: `revoked_at IS NULL` и `period_start ==` текущий `period_start` пользователя (как ключ ledger).
- Комментарий опционален; отзыв обязателен в продукте.
- Auth API: `require_superuser` (как выдача подписки).
- Безлимит (`max_mh_credits_per_period is None`) — grant отклоняется.
- Нет `any` / `Record<string, any>`; UI без внутренних имён сервисов/SQL.
- Коммитить только по явной просьбе пользователя.
- Тесты backend: `cd backend && uv run pytest <path> -v`.
- Линтер backend: `cd backend && uv run ruff check <paths>`.
- Перед правками кода — согласование с пользователем по правилу репо (если сессия новая).

## File map

Создать:

- `backend/src/markethacker/modules/billing/domain/mh_credits_admin_grant.py`
- `backend/alembic/versions/20261109_0062_mh_credits_admin_grants.py`
- `backend/src/markethacker/modules/billing/infrastructure/mh_credits_admin_grant_repository.py`
- `backend/src/markethacker/modules/billing/application/mh_credits_admin_grant_service.py`
- `backend/tests/unit/billing/test_mh_credits_admin_grants.py`
- `backend/tests/integration/test_admin_mh_credits_grants.py`
- `admin-panel/src/components/mh-credits-panel.tsx`

Изменить:

- `backend/src/markethacker/infrastructure/database/all_models.py`
- `backend/src/markethacker/modules/billing/application/limit_adjustments.py`
- `backend/src/markethacker/modules/billing/application/service.py` (`_apply_limit_boosts`)
- `backend/src/markethacker/modules/admin/api/schemas.py`
- `backend/src/markethacker/modules/admin/api/router.py`
- `admin-panel/src/lib/api.ts`
- `admin-panel/src/app/(admin)/users/[id]/page.tsx`
- `docs/architecture/billing.md`
- `docs/superpowers/specs/2026-11-09-admin-mh-credits-grant-design.md` (статус → approved)

---

### Task 1: ORM + миграция

**Files:**
- Create: `backend/src/markethacker/modules/billing/domain/mh_credits_admin_grant.py`
- Create: `backend/alembic/versions/20261109_0062_mh_credits_admin_grants.py`
- Modify: `backend/src/markethacker/infrastructure/database/all_models.py`

**Interfaces:**
- Produces:
  - `BillingMhCreditsAdminGrant` ORM
  - table `billing_mh_credits_admin_grants`
  - alembic `revision = "20261109_0062"`, `down_revision = "20261018_0061"`

- [ ] **Step 1: Create ORM model**

```python
"""ORM: admin-начисление Мурликов на billing-период."""

from __future__ import annotations

import uuid
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Index, Integer, Text, text
from sqlalchemy.orm import Mapped, mapped_column

from markethacker.infrastructure.database.base import Base, TimestampMixin


class BillingMhCreditsAdminGrant(TimestampMixin, Base):
    """Разовое увеличение лимита Мурликов админом на один billing-период."""

    __tablename__ = "billing_mh_credits_admin_grants"
    __table_args__ = (
        Index("ix_billing_mh_credits_admin_grants_user", "user_id"),
        Index(
            "ix_billing_mh_credits_admin_grants_user_period",
            "user_id",
            "period_start",
        ),
    )

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("users.id", ondelete="CASCADE"),
        nullable=False,
    )
    units: Mapped[int] = mapped_column(Integer, nullable=False)
    period_start: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    period_end: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    comment: Mapped[str | None] = mapped_column(Text, nullable=True)
    granted_by_admin_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("users.id", ondelete="RESTRICT"),
        nullable=False,
    )
    revoked_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
    )
    revoked_by_admin_id: Mapped[uuid.UUID | None] = mapped_column(
        ForeignKey("users.id", ondelete="SET NULL"),
        nullable=True,
    )
    revoke_comment: Mapped[str | None] = mapped_column(Text, nullable=True)
```

`TimestampMixin` уже даёт `created_at` / `updated_at`. Не дублировать `created_at`.

- [ ] **Step 2: Alembic migration**

```python
"""Admin grants of MH credits per billing period.

Revision ID: 20261109_0062
Revises: 20261018_0061
Create Date: 2026-11-09
"""

from __future__ import annotations

import sqlalchemy as sa
from alembic import op

revision = "20261109_0062"
down_revision = "20261018_0061"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "billing_mh_credits_admin_grants",
        sa.Column("id", sa.UUID(), nullable=False),
        sa.Column("user_id", sa.UUID(), nullable=False),
        sa.Column("units", sa.Integer(), nullable=False),
        sa.Column("period_start", sa.DateTime(timezone=True), nullable=False),
        sa.Column("period_end", sa.DateTime(timezone=True), nullable=False),
        sa.Column("comment", sa.Text(), nullable=True),
        sa.Column("granted_by_admin_id", sa.UUID(), nullable=False),
        sa.Column("revoked_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("revoked_by_admin_id", sa.UUID(), nullable=True),
        sa.Column("revoke_comment", sa.Text(), nullable=True),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.Column(
            "updated_at",
            sa.DateTime(timezone=True),
            server_default=sa.text("now()"),
            nullable=False,
        ),
        sa.CheckConstraint("units > 0", name="ck_billing_mh_credits_admin_grants_units_positive"),
        sa.ForeignKeyConstraint(["user_id"], ["users.id"], ondelete="CASCADE"),
        sa.ForeignKeyConstraint(["granted_by_admin_id"], ["users.id"], ondelete="RESTRICT"),
        sa.ForeignKeyConstraint(["revoked_by_admin_id"], ["users.id"], ondelete="SET NULL"),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index(
        "ix_billing_mh_credits_admin_grants_user",
        "billing_mh_credits_admin_grants",
        ["user_id"],
    )
    op.create_index(
        "ix_billing_mh_credits_admin_grants_user_period",
        "billing_mh_credits_admin_grants",
        ["user_id", "period_start"],
    )


def downgrade() -> None:
    op.drop_index(
        "ix_billing_mh_credits_admin_grants_user_period",
        table_name="billing_mh_credits_admin_grants",
    )
    op.drop_index(
        "ix_billing_mh_credits_admin_grants_user",
        table_name="billing_mh_credits_admin_grants",
    )
    op.drop_table("billing_mh_credits_admin_grants")
```

Сверить фактические имена колонок `TimestampMixin` / соседние миграции (`created_at`/`updated_at` server_default) перед финализацией.

- [ ] **Step 3: Register in `all_models.py`**

Рядом с импортом ledger:

```python
from markethacker.modules.billing.domain.mh_credits_admin_grant import (  # noqa: F401
    BillingMhCreditsAdminGrant,
)
```

- [ ] **Step 4: Commit (только если пользователь попросил)**

```bash
cd backend
git add src/markethacker/modules/billing/domain/mh_credits_admin_grant.py \
  alembic/versions/20261109_0062_mh_credits_admin_grants.py \
  src/markethacker/infrastructure/database/all_models.py
git commit -m "feat(billing): add mh credits admin grant table"
```

---

### Task 2: Aggregation в effective plan + unit-тесты

**Files:**
- Create: `backend/src/markethacker/modules/billing/infrastructure/mh_credits_admin_grant_repository.py`
- Modify: `backend/src/markethacker/modules/billing/application/limit_adjustments.py`
- Modify: `backend/src/markethacker/modules/billing/application/service.py`
- Create: `backend/tests/unit/billing/test_mh_credits_admin_grants.py`

**Interfaces:**
- Consumes: `BillingMhCreditsAdminGrant`, `merge_limit_extras`, `apply_limit_extras_to_plan`
- Produces:
  - `aggregate_admin_mh_credits_grants(grants: list[BillingMhCreditsAdminGrant]) -> dict[str, int]`
  - `MhCreditsAdminGrantRepository.sum_active_units(user_id, period_start) -> int`
  - `MhCreditsAdminGrantRepository.list_for_period(user_id, period_start) -> list[BillingMhCreditsAdminGrant]`
  - `_apply_limit_boosts` учитывает admin grants

- [ ] **Step 1: Write failing unit tests**

`backend/tests/unit/billing/test_mh_credits_admin_grants.py`:

```python
"""Admin grants → effective MH credits limit."""

from __future__ import annotations

import uuid
from datetime import UTC, datetime, timedelta
from types import SimpleNamespace

from markethacker.modules.billing.application.limit_adjustments import (
    aggregate_admin_mh_credits_grants,
    apply_limit_extras_to_plan,
    merge_limit_extras,
)
from markethacker.modules.billing.domain.models import BillingPlan


def _grant(*, units: int, revoked: bool = False) -> SimpleNamespace:
    return SimpleNamespace(
        units=units,
        revoked_at=datetime.now(UTC) if revoked else None,
    )


def test_aggregate_admin_grants_sums_active_only() -> None:
    totals = aggregate_admin_mh_credits_grants(
        [_grant(units=40), _grant(units=10), _grant(units=5, revoked=True)]
    )
    assert totals["mh_credits"] == 50


def test_apply_admin_grants_increases_plan_limit() -> None:
    plan = BillingPlan(
        id=uuid.uuid4(),
        name="pro",
        display_name="Pro",
        price_monthly=1990,
        price_yearly=None,
        max_members=5,
        max_organizations=1,
        max_api_calls_per_day=1000,
        max_mh_credits_per_period=100,
        max_reviews_products_per_period=10,
        features=[],
        marketing=None,
        is_active=True,
        stripe_price_id=None,
        stripe_price_yearly_id=None,
        yookassa_item_id=None,
    )
    extras = merge_limit_extras(aggregate_admin_mh_credits_grants([_grant(units=25)]))
    adjusted = apply_limit_extras_to_plan(plan, extras)
    assert adjusted.max_mh_credits_per_period == 125
    assert adjusted.max_reviews_products_per_period == 12
```

Подправить конструктор `BillingPlan` под фактические обязательные поля модели (смотреть `domain/models.py` / соседние тесты `test_limit_addons.py`).

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd backend && uv run pytest tests/unit/billing/test_mh_credits_admin_grants.py -v
```

Expected: FAIL — `aggregate_admin_mh_credits_grants` не найден.

- [ ] **Step 3: Implement `aggregate_admin_mh_credits_grants` in `limit_adjustments.py`**

```python
def aggregate_admin_mh_credits_grants(grants: list[Any]) -> dict[str, int]:
    """Суммирует активные admin-гранты Мурликов (revoked_at is None)."""
    totals = {defn.key: 0 for defn in LIMIT_CATALOG.values()}
    total = 0
    for grant in grants:
        if getattr(grant, "revoked_at", None) is not None:
            continue
        units = int(getattr(grant, "units", 0) or 0)
        if units > 0:
            total += units
    totals["mh_credits"] = total
    return totals
```

Добавить нужный import/`Any` в TYPE_CHECKING по стилю файла.

- [ ] **Step 4: Repository**

```python
"""Репозиторий admin-грантов Мурликов."""

from __future__ import annotations

import uuid
from datetime import datetime

from sqlalchemy import Select, func, select
from sqlalchemy.ext.asyncio import AsyncSession

from markethacker.modules.billing.domain.mh_credits_admin_grant import (
    BillingMhCreditsAdminGrant,
)


class MhCreditsAdminGrantRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    def _active_for_period_stmt(
        self,
        user_id: uuid.UUID,
        period_start: datetime,
    ) -> Select[tuple[BillingMhCreditsAdminGrant]]:
        return select(BillingMhCreditsAdminGrant).where(
            BillingMhCreditsAdminGrant.user_id == user_id,
            BillingMhCreditsAdminGrant.period_start == period_start,
            BillingMhCreditsAdminGrant.revoked_at.is_(None),
        )

    async def sum_active_units(
        self,
        user_id: uuid.UUID,
        period_start: datetime,
    ) -> int:
        value = await self._session.scalar(
            select(func.coalesce(func.sum(BillingMhCreditsAdminGrant.units), 0)).where(
                BillingMhCreditsAdminGrant.user_id == user_id,
                BillingMhCreditsAdminGrant.period_start == period_start,
                BillingMhCreditsAdminGrant.revoked_at.is_(None),
            )
        )
        return int(value or 0)

    async def list_for_period(
        self,
        user_id: uuid.UUID,
        period_start: datetime,
    ) -> list[BillingMhCreditsAdminGrant]:
        rows = await self._session.scalars(
            select(BillingMhCreditsAdminGrant)
            .where(
                BillingMhCreditsAdminGrant.user_id == user_id,
                BillingMhCreditsAdminGrant.period_start == period_start,
            )
            .order_by(BillingMhCreditsAdminGrant.created_at.desc())
        )
        return list(rows)

    async def list_active_for_period(
        self,
        user_id: uuid.UUID,
        period_start: datetime,
    ) -> list[BillingMhCreditsAdminGrant]:
        rows = await self._session.scalars(
            self._active_for_period_stmt(user_id, period_start).order_by(
                BillingMhCreditsAdminGrant.created_at.desc()
            )
        )
        return list(rows)

    async def get_for_user(
        self,
        grant_id: uuid.UUID,
        user_id: uuid.UUID,
    ) -> BillingMhCreditsAdminGrant | None:
        return await self._session.scalar(
            select(BillingMhCreditsAdminGrant).where(
                BillingMhCreditsAdminGrant.id == grant_id,
                BillingMhCreditsAdminGrant.user_id == user_id,
            )
        )

    async def add(self, grant: BillingMhCreditsAdminGrant) -> BillingMhCreditsAdminGrant:
        self._session.add(grant)
        await self._session.flush()
        return grant
```

- [ ] **Step 5: Wire into `_apply_limit_boosts`**

В `BillingService._apply_limit_boosts` заменить early-return и merge:

```python
async def _apply_limit_boosts(self, user_id: uuid.UUID, plan: BillingPlan) -> BillingPlan:
    from markethacker.modules.billing.application.limit_adjustments import (
        aggregate_admin_mh_credits_grants,
        aggregate_promo_boosts,
        aggregate_purchased_addons,
        apply_limit_extras_to_plan,
        merge_limit_extras,
    )
    from markethacker.modules.billing.application.mh_credits_service import MhCreditsService
    from markethacker.modules.billing.infrastructure.limit_addon_repository import (
        LimitAddonRepository,
    )
    from markethacker.modules.billing.infrastructure.mh_credits_admin_grant_repository import (
        MhCreditsAdminGrantRepository,
    )
    from markethacker.modules.billing.infrastructure.promo_repository import PromoRepository

    promo_rows = await PromoRepository(self._session).get_active_limit_boosts(user_id)
    addon_rows = await LimitAddonRepository(self._session).get_active_entitlements(user_id)
    period_start, _period_end = await MhCreditsService(self._session)._period_for_user(user_id)
    admin_grants = await MhCreditsAdminGrantRepository(self._session).list_active_for_period(
        user_id,
        period_start,
    )

    if not promo_rows and not addon_rows and not admin_grants:
        return plan

    extras = merge_limit_extras(
        aggregate_promo_boosts(promo_rows),
        aggregate_purchased_addons(addon_rows),
        aggregate_admin_mh_credits_grants(admin_grants),
    )
    return apply_limit_extras_to_plan(plan, extras)
```

Избежать циклического импорта: если `MhCreditsService` → `BillingService` создаёт цикл при импорте метода периода, вынести `_period_for_user` в маленький helper (например `billing/application/period.py`) или продублировать чтение subscription period в `_apply_limit_boosts` так же, как в `MhCreditsService._period_for_user` (предпочтительно inline-копия 5 строк / private method на `BillingService`, без нового публичного API).

Предпочтительный вариант без цикла:

```python
async def _billing_period_for_user(self, user_id: uuid.UUID) -> tuple[datetime, datetime]:
    sub = await self.get_subscription(user_id)
    if sub is None:
        now = datetime.now(UTC)
        return now.replace(day=1, hour=0, minute=0, second=0, microsecond=0), now
    return sub.current_period_start, sub.current_period_end
```

И использовать его и из `MhCreditsService` (опциональный рефактор в том же PR только если правка минимальна; иначе дублировать в `_apply_limit_boosts`).

- [ ] **Step 6: Run unit tests — expect PASS**

```bash
cd backend && uv run pytest tests/unit/billing/test_mh_credits_admin_grants.py -v
```

Expected: PASS

- [ ] **Step 7: Commit (если пользователь попросил)**

```bash
cd backend
git add src/markethacker/modules/billing/application/limit_adjustments.py \
  src/markethacker/modules/billing/application/service.py \
  src/markethacker/modules/billing/infrastructure/mh_credits_admin_grant_repository.py \
  tests/unit/billing/test_mh_credits_admin_grants.py
git commit -m "feat(billing): apply admin mh credit grants to effective plan"
```

---

### Task 3: Admin grant service + API + integration tests

**Files:**
- Create: `backend/src/markethacker/modules/billing/application/mh_credits_admin_grant_service.py`
- Modify: `backend/src/markethacker/modules/admin/api/schemas.py`
- Modify: `backend/src/markethacker/modules/admin/api/router.py`
- Create: `backend/tests/integration/test_admin_mh_credits_grants.py`

**Interfaces:**
- Consumes: repository, `BillingService.get_effective_plan`, `MhCreditsService.status`, `effective_plan_cache.invalidate`, `AuditLogRepository.log`
- Produces:
  - `MhCreditsAdminGrantService.get_status(user_id) -> AdminMhCreditsStatus`
  - `MhCreditsAdminGrantService.grant(user_id, admin_id, units, comment, ip) -> BillingMhCreditsAdminGrant`
  - `MhCreditsAdminGrantService.revoke(user_id, grant_id, admin_id, comment, ip) -> BillingMhCreditsAdminGrant`
  - Routes:
    - `GET /admin/users/{user_id}/mh-credits`
    - `POST /admin/users/{user_id}/mh-credits/grants`
    - `POST /admin/users/{user_id}/mh-credits/grants/{grant_id}/revoke`

- [ ] **Step 1: Schemas in `admin/api/schemas.py`**

```python
class AdminMhCreditsGrantItem(CamelModel):
    id: uuid.UUID
    units: int
    comment: str | None
    created_at: datetime
    granted_by_admin_id: uuid.UUID
    revoked_at: datetime | None
    revoked_by_admin_id: uuid.UUID | None
    revoke_comment: str | None
    period_start: datetime
    period_end: datetime


class AdminMhCreditsStatusResponse(CamelModel):
    limit: int | None
    used: int
    remaining: int | None
    period_start: datetime
    period_end: datetime
    admin_grant_units: int
    grants: list[AdminMhCreditsGrantItem]


class AdminMhCreditsGrantRequest(CamelModel):
    units: int = Field(gt=0, le=1_000_000)
    comment: str | None = Field(default=None, max_length=2000)


class AdminMhCreditsRevokeRequest(CamelModel):
    comment: str | None = Field(default=None, max_length=2000)
```

- [ ] **Step 2: Service**

```python
"""Admin grant / revoke Мурликов на текущий billing-период."""

from __future__ import annotations

import uuid
from dataclasses import dataclass
from datetime import UTC, datetime

from sqlalchemy.ext.asyncio import AsyncSession

from markethacker.modules.auth.infrastructure.audit_repository import AuditLogRepository
from markethacker.modules.billing.application.mh_credits_service import MhCreditsService
from markethacker.modules.billing.application.service import BillingService
from markethacker.modules.billing.domain.mh_credits_admin_grant import (
    BillingMhCreditsAdminGrant,
)
from markethacker.modules.billing.infrastructure.cached import (
    effective_plan as effective_plan_cache,
)
from markethacker.modules.billing.infrastructure.mh_credits_admin_grant_repository import (
    MhCreditsAdminGrantRepository,
)
from markethacker.modules.reviews_analysis.domain.quota_mode import QUOTA_MODE_PER_MEMBER_SEAT
from markethacker.modules.users.domain.models import User
from markethacker.modules.users.infrastructure.repository import UserRepository
from markethacker.shared.exceptions import NotFoundError, ValidationError


@dataclass(frozen=True, slots=True)
class AdminMhCreditsStatus:
    limit: int | None
    used: int
    remaining: int | None
    period_start: datetime
    period_end: datetime
    admin_grant_units: int
    grants: list[BillingMhCreditsAdminGrant]


class MhCreditsAdminGrantService:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session
        self._repo = MhCreditsAdminGrantRepository(session)
        self._billing = BillingService(session)
        self._credits = MhCreditsService(session)
        self._audit = AuditLogRepository(session)

    async def get_status(self, user_id: uuid.UUID) -> AdminMhCreditsStatus:
        await self._require_user(user_id)
        status = await self._credits.status(
            user_id=user_id,
            quota_mode=QUOTA_MODE_PER_MEMBER_SEAT,
            organization_id=None,
            prefer_personal=True,
        )
        grants = await self._repo.list_for_period(user_id, status.period_start)
        admin_units = sum(g.units for g in grants if g.revoked_at is None)
        return AdminMhCreditsStatus(
            limit=status.limit,
            used=status.used,
            remaining=status.remaining,
            period_start=status.period_start,
            period_end=status.period_end,
            admin_grant_units=admin_units,
            grants=grants,
        )

    async def grant(
        self,
        *,
        user_id: uuid.UUID,
        admin_id: uuid.UUID,
        units: int,
        comment: str | None,
        ip_address: str | None,
    ) -> BillingMhCreditsAdminGrant:
        await self._require_user(user_id)
        if units < 1:
            raise ValidationError("Количество Мурликов должно быть больше нуля.")

        plan = await self._billing.get_effective_plan(user_id)
        if plan.max_mh_credits_per_period is None:
            raise ValidationError("У пользователя безлимитный тариф — начисление не нужно.")

        period_start, period_end = await self._grant_period_window(user_id)

        grant = BillingMhCreditsAdminGrant(
            user_id=user_id,
            units=units,
            period_start=period_start,
            period_end=period_end,
            comment=comment,
            granted_by_admin_id=admin_id,
        )
        await self._repo.add(grant)
        await effective_plan_cache.invalidate(user_id)

        await self._audit.log(
            action="admin.mh_credits.granted",
            user_id=admin_id,
            org_id=None,
            resource_type="billing_mh_credits_admin_grant",
            resource_id=grant.id,
            ip_address=ip_address,
            metadata={
                "target_user_id": str(user_id),
                "units": units,
                "comment": comment,
                "period_start": period_start.isoformat(),
                "period_end": period_end.isoformat(),
            },
        )
        return grant

    async def revoke(
        self,
        *,
        user_id: uuid.UUID,
        grant_id: uuid.UUID,
        admin_id: uuid.UUID,
        comment: str | None,
        ip_address: str | None,
    ) -> BillingMhCreditsAdminGrant:
        await self._require_user(user_id)
        grant = await self._repo.get_for_user(grant_id, user_id)
        if grant is None:
            raise NotFoundError("BillingMhCreditsAdminGrant", grant_id)
        if grant.revoked_at is not None:
            raise ValidationError("Начисление уже отозвано.")

        grant.revoked_at = datetime.now(UTC)
        grant.revoked_by_admin_id = admin_id
        grant.revoke_comment = comment
        await self._session.flush()
        await effective_plan_cache.invalidate(user_id)

        await self._audit.log(
            action="admin.mh_credits.revoked",
            user_id=admin_id,
            org_id=None,
            resource_type="billing_mh_credits_admin_grant",
            resource_id=grant.id,
            ip_address=ip_address,
            metadata={
                "target_user_id": str(user_id),
                "units": grant.units,
                "comment": comment,
            },
        )
        return grant

    async def _require_user(self, user_id: uuid.UUID) -> User:
        user = await UserRepository(self._session).get_by_id(user_id)
        if user is None:
            raise NotFoundError("User", user_id)
        return user

    async def _grant_period_window(self, user_id: uuid.UUID) -> tuple[datetime, datetime]:
        sub = await self._billing.get_subscription(user_id)
        if sub is not None:
            return sub.current_period_start, sub.current_period_end
        now = datetime.now(UTC)
        period_start = now.replace(day=1, hour=0, minute=0, second=0, microsecond=0)
        if period_start.month == 12:
            period_end = period_start.replace(year=period_start.year + 1, month=1)
        else:
            period_end = period_start.replace(month=period_start.month + 1)
        return period_start, period_end
```

`sum_active` / effective plan матчат только по `period_start` (совпадает с ledger и с `_period_for_user` у free). `period_end` — для отображения и аудита; у free без подписки — конец календарного месяца, не `now`.

Сигнатуру `AuditLogRepository.log` брать как в `AdminService.grant_subscription`.

- [ ] **Step 3: Router endpoints**

Рядом с user routes в `admin/api/router.py`:

```python
@router.get(
    "/users/{user_id}/mh-credits",
    response_model=APIResponse[AdminMhCreditsStatusResponse],
    summary="Остаток Мурликов и admin-гранты",
)
async def get_user_mh_credits(
    user_id: uuid.UUID,
    session: Annotated[AsyncSession, Depends(get_admin_db)],
    _admin_id: Annotated[str, Depends(require_superuser)],
) -> APIResponse[AdminMhCreditsStatusResponse]:
    status = await MhCreditsAdminGrantService(session).get_status(user_id)
    return APIResponse(data=_mh_credits_status_response(status))


@router.post(
    "/users/{user_id}/mh-credits/grants",
    response_model=APIResponse[AdminMhCreditsGrantItem],
    status_code=201,
    summary="Начислить Мурлики на текущий период",
)
async def grant_user_mh_credits(
    user_id: uuid.UUID,
    body: AdminMhCreditsGrantRequest,
    request: Request,
    session: Annotated[AsyncSession, Depends(get_admin_db)],
    admin_id: Annotated[str, Depends(require_superuser)],
) -> APIResponse[AdminMhCreditsGrantItem]:
    grant = await MhCreditsAdminGrantService(session).grant(
        user_id=user_id,
        admin_id=uuid.UUID(admin_id),
        units=body.units,
        comment=body.comment,
        ip_address=request.client.host if request.client else None,
    )
    await session.commit()  # только если соседние admin POST сами коммитят — иначе следовать паттерну get_admin_db
    return APIResponse(data=_mh_credits_grant_item(grant))


@router.post(
    "/users/{user_id}/mh-credits/grants/{grant_id}/revoke",
    response_model=APIResponse[AdminMhCreditsGrantItem],
    summary="Отозвать начисление Мурликов",
)
async def revoke_user_mh_credits_grant(
    user_id: uuid.UUID,
    grant_id: uuid.UUID,
    body: AdminMhCreditsRevokeRequest,
    request: Request,
    session: Annotated[AsyncSession, Depends(get_admin_db)],
    admin_id: Annotated[str, Depends(require_superuser)],
) -> APIResponse[AdminMhCreditsGrantItem]:
    grant = await MhCreditsAdminGrantService(session).revoke(
        user_id=user_id,
        grant_id=grant_id,
        admin_id=uuid.UUID(admin_id),
        comment=body.comment,
        ip_address=request.client.host if request.client else None,
    )
    return APIResponse(data=_mh_credits_grant_item(grant))
```

Мапперы `_mh_credits_grant_item` / `_mh_credits_status_response` — локальные функции в router, как `_subscription_response`.  
Коммит сессии — **строго по паттерну соседних admin POST** (многие полагаются на middleware/`get_admin_db`).

- [ ] **Step 4: Integration test**

Паттерн `_admin_headers` из `tests/integration/test_limit_addons.py`.

Сценарий:

1. Зарегистрировать обычного user (или взять fixture), выдать ему подписку с конечным `max_mh_credits_per_period` через admin grant subscription / DB.
2. `GET /api/v1/admin/users/{id}/mh-credits` → 200, запомнить `remaining`.
3. `POST .../grants` `{ "units": 30 }` → 201.
4. Повторный GET → `remaining` +30, `adminGrantUnits` ≥ 30.
5. `POST .../grants/{id}/revoke` → 200; GET → remaining вернулся.
6. Повторный revoke → 4xx.
7. (опционально) безлимитный план → POST grant → 4xx.

```bash
cd backend && uv run pytest tests/integration/test_admin_mh_credits_grants.py -v
```

Expected: PASS

- [ ] **Step 5: Ruff**

```bash
cd backend && uv run ruff check \
  src/markethacker/modules/billing/application/mh_credits_admin_grant_service.py \
  src/markethacker/modules/admin/api/router.py \
  src/markethacker/modules/admin/api/schemas.py \
  tests/integration/test_admin_mh_credits_grants.py
```

- [ ] **Step 6: Commit (если пользователь попросил)**

```bash
cd backend
git add src/markethacker/modules/billing/application/mh_credits_admin_grant_service.py \
  src/markethacker/modules/admin/api/schemas.py \
  src/markethacker/modules/admin/api/router.py \
  tests/integration/test_admin_mh_credits_grants.py
git commit -m "feat(admin): grant and revoke mh credits for user"
```

---

### Task 4: Admin Panel UI

**Files:**
- Create: `admin-panel/src/components/mh-credits-panel.tsx`
- Modify: `admin-panel/src/lib/api.ts`
- Modify: `admin-panel/src/app/(admin)/users/[id]/page.tsx`

**Interfaces:**
- Consumes: admin API из Task 3
- Produces: `MhCreditsPanel` на карточке пользователя (видно superuser, как блок подписки)

- [ ] **Step 1: Types in `lib/api.ts`**

```typescript
export interface AdminMhCreditsGrant {
  id: string;
  units: number;
  comment: string | null;
  createdAt: string;
  grantedByAdminId: string;
  revokedAt: string | null;
  revokedByAdminId: string | null;
  revokeComment: string | null;
  periodStart: string;
  periodEnd: string;
}

export interface AdminMhCreditsStatus {
  limit: number | null;
  used: number;
  remaining: number | null;
  periodStart: string;
  periodEnd: string;
  adminGrantUnits: number;
  grants: AdminMhCreditsGrant[];
}
```

- [ ] **Step 2: Component `mh-credits-panel.tsx`**

Поведение:

- `useApi` → `GET /admin/users/${userId}/mh-credits`
- Показать период (`formatDate`), лимит / использовано / остаток, вклад admin (`adminGrantUnits`)
- Если `limit === null` — текст «Лимит не ограничен», без кнопки «Начислить»
- Иначе кнопка «Начислить» → Modal: number input `units` (>0), optional comment → POST grants → toast «Начислено» / «Не удалось начислить Мурлики» → reload
- Таблица `grants`: units, comment, createdAt, статус (Активно / Отозвано), кнопка «Отозвать» только если `revokedAt == null`
- Отзыв: ConfirmDialog + optional comment → POST revoke → toast «Отозвано»

Использовать `MurlikIcon`, `Card` не обязательно снаружи (родитель обернёт), либо сам `Card title="Мурлики"`.

Паттерны: `useMutation`, `Modal`, `ConfirmDialog`, `useToast` как в `grant-subscription-modal.tsx`.

- [ ] **Step 3: Embed in `users/[id]/page.tsx`**

После карточки «Подписка», внутри условия `isSuper` (или рядом с транзакциями):

```tsx
{isSuper && <MhCreditsPanel userId={user.id} />}
```

- [ ] **Step 4: Smoke**

Вручную: открыть пользователя с конечным лимитом → начислить → остаток вырос → отозвать → остаток упал.  
На безлимите кнопки нет.

- [ ] **Step 5: Commit (если пользователь попросил)**

```bash
cd admin-panel
git add src/components/mh-credits-panel.tsx src/lib/api.ts src/app/\(admin\)/users/\[id\]/page.tsx
git commit -m "feat(admin-panel): mh credits grant panel on user page"
```

---

### Task 5: Документация

**Files:**
- Modify: `docs/architecture/billing.md`
- Modify: `docs/superpowers/specs/2026-11-09-admin-mh-credits-grant-design.md`

- [ ] **Step 1: Update `billing.md` § Применение лимитов**

Заменить формулу:

```text
effective_limit = plan_limit + promo_boosts + purchased_addons + admin_mh_credits_grants
```

Краткий абзац: admin-гранты в `billing_mh_credits_admin_grants`, активны для текущего `period_start`, отзыв через админку карточки пользователя; ledger `used` не меняется.  
Инвалидация кэша effective plan при grant/revoke.

- [ ] **Step 2: Spec status → `approved`**

- [ ] **Step 3: Commit docs (если пользователь попросил)**

```bash
cd docs
git add architecture/billing.md superpowers/specs/2026-11-09-admin-mh-credits-grant-design.md
git commit -m "docs: admin mh credits grants in billing model"
```

---

## Spec coverage checklist

| Spec requirement | Task |
|---|---|
| Таблица admin-grant | Task 1 |
| Учёт в effective plan | Task 2 |
| GET status + grants | Task 3 |
| POST grant | Task 3 |
| POST revoke | Task 3 |
| Audit actions | Task 3 |
| Unlimited reject | Task 3 |
| UI блок / modal / revoke | Task 4 |
| Unit + API tests | Task 2–3 |
| billing.md | Task 5 |
| Owner→org ceiling via effective plan | Task 2 (без отдельного кода) |

## Consistency notes

- Активность гранта для лимита — по `period_start`, не по `now < period_end` (совместимо с ledger и free-пользователями).
- `period_end` при grant для free нормализуется до конца календарного месяца.
- Коммиты — только по просьбе пользователя.
