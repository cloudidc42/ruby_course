# Part 094: Capstone 2 — SaaS Multi-Tenant App: Team, Organization, Permission, Invite Flow

> **Step ครอบคลุมใน Part นี้:** Step 931–940

**ระดับ:** Advanced | **Ruby:** 3.2+ | **Rails:** 7.1+ | **PostgreSQL:** 15+

ต่อเนื่องจาก Part 093 ที่เราวางรากฐาน SaaS App และ Billing ระบบแล้ว ใน Part นี้เราจะสร้างระบบ Team Management ที่สมบูรณ์แบบ ครอบคลุมตั้งแต่ Membership Roles, Pundit Authorization สำหรับ Multi-tenant, ระบบ Invite Flow ที่ปลอดภัย, Organization Settings, Audit Log, Account Switching, API Access, Onboarding Wizard และ SaaS Metrics Dashboard ทุก component เหล่านี้ล้วนจำเป็นสำหรับ SaaS Application ระดับ Production

---

## สารบัญ

- [Step 931: Team Model — Membership Join Table](#step-931-team-model-membership-join-table)
- [Step 932: Pundit Policies สำหรับ Multi-tenant](#step-932-pundit-policies-สำหรับ-multi-tenant)
- [Step 933: Invite Flow — Invitation Model](#step-933-invite-flow-invitation-model)
- [Step 934: Accept Invitation](#step-934-accept-invitation)
- [Step 935: Organization Settings Page](#step-935-organization-settings-page)
- [Step 936: Audit Log](#step-936-audit-log)
- [Step 937: Account Switching](#step-937-account-switching)
- [Step 938: API Access สำหรับ SaaS](#step-938-api-access-สำหรับ-saas)
- [Step 939: Onboarding Flow](#step-939-onboarding-flow)
- [Step 940: SaaS Metrics Dashboard](#step-940-saas-metrics-dashboard)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุปสิ่งที่ได้เรียนรู้)

---

## Step 931: Team Model — Membership Join Table

Membership เป็น join table ที่เชื่อม User กับ Account โดยมี Role กำหนดสิทธิ์การใช้งาน

### Schema Design

```ruby
# Membership มี 3 roles หลัก:
# owner   — สร้าง account, จัดการ billing, ลบ account ได้
# admin   — invite/remove members, จัดการ settings
# member  — ใช้งาน features ตาม plan
```

### Migration สำหรับ Membership

```bash
rails generate model Membership \
  user:references \
  account:references \
  role:string \
  invited_by:references \
  joined_at:datetime \
  last_active_at:datetime
```

```ruby
# db/migrate/xxx_create_memberships.rb
class CreateMemberships < ActiveRecord::Migration[7.1]
  def change
    create_table :memberships do |t|
      t.references :user, null: false, foreign_key: true
      t.references :account, null: false, foreign_key: true
      t.string :role, null: false, default: "member"
      t.references :invited_by, foreign_key: { to_table: :users }
      t.datetime :joined_at
      t.datetime :last_active_at

      t.timestamps
    end

    # User อยู่ได้แค่ account ละ 1 membership
    add_index :memberships, [:user_id, :account_id], unique: true
    add_index :memberships, [:account_id, :role]
    add_index :memberships, :last_active_at
  end
end
```

### Membership Model

```ruby
# app/models/membership.rb
class Membership < ApplicationRecord
  # Associations
  belongs_to :user
  belongs_to :account
  belongs_to :invited_by, class_name: "User", optional: true

  # Role enum
  ROLES = %w[owner admin member].freeze

  validates :role, inclusion: { in: ROLES }
  validates :user_id, uniqueness: {
    scope: :account_id,
    message: "ผู้ใช้นี้เป็นสมาชิกของ account นี้อยู่แล้ว"
  }

  # Scopes
  scope :owners, -> { where(role: "owner") }
  scope :admins, -> { where(role: "admin") }
  scope :members, -> { where(role: "member") }
  scope :active, -> { where.not(last_active_at: nil) }
  scope :recently_active, -> { where(last_active_at: 30.days.ago..) }

  # Role helpers
  def owner?
    role == "owner"
  end

  def admin?
    role == "admin"
  end

  def member?
    role == "member"
  end

  # owner หรือ admin ถือว่ามี admin permission
  def admin_or_owner?
    admin? || owner?
  end

  def display_role
    I18n.t("roles.#{role}", default: role.humanize)
  end

  # อัปเดต last_active_at
  def touch_activity!
    update_column(:last_active_at, Time.current)
  end

  # ไม่สามารถ remove owner ที่เหลืออยู่คนเดียว
  def removable?
    return true unless owner?
    account.memberships.owners.count > 1
  end
end
```

### User Model Extensions

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :trackable

  has_many :memberships, dependent: :destroy
  has_many :accounts, through: :memberships
  has_many :sent_invitations, class_name: "Invitation", foreign_key: :inviter_id

  validates :name, presence: true, length: { minimum: 2, maximum: 100 }

  def membership_for(account)
    memberships.find_by(account: account)
  end

  def role_in(account)
    membership_for(account)&.role
  end

  def owner_of?(account)
    role_in(account) == "owner"
  end

  def admin_of?(account)
    role_in(account).in?(%w[admin owner])
  end

  def member_of?(account)
    memberships.exists?(account: account)
  end

  # Format สำหรับ display
  def display_name
    name.presence || email.split("@").first
  end

  def initials
    display_name.split.map(&:first).join.upcase.first(2)
  end
end
```

---

## Step 932: Pundit Policies สำหรับ Multi-tenant

Pundit policies ต้องออกแบบให้ตระหนักถึง multi-tenancy — ทุก authorization check ต้องรวม tenant context ด้วย

### Application Policy Base

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :current, :record

  def initialize(current, record)
    # current คือ Current object (ไม่ใช่แค่ user)
    # ทำให้เราเข้าถึงได้ทั้ง current.user และ current.account
    @current = current
    @record = record

    raise Pundit::NotAuthorizedError, "ต้อง login ก่อน" unless current.user
  end

  # Helper methods ที่ใช้บ่อย
  def user
    current.user
  end

  def account
    current.account
  end

  def membership
    current.membership
  end

  def owner?
    membership&.owner?
  end

  def admin?
    membership&.admin_or_owner?
  end

  def member?
    membership.present?
  end

  # ตรวจสอบว่า record เป็นของ current account
  def belongs_to_current_account?
    record.respond_to?(:account_id) && record.account_id == account&.id
  end

  class Scope
    def initialize(current, scope)
      @current = current
      @scope = scope
      @account = current.account
    end

    def resolve
      raise NotImplementedError, "Implement resolve in #{self.class}"
    end

    private

    attr_reader :current, :scope, :account

    def owner?
      current.membership&.owner?
    end

    def admin?
      current.membership&.admin_or_owner?
    end
  end
end
```

### AccountPolicy

```ruby
# app/policies/account_policy.rb
class AccountPolicy < ApplicationPolicy
  # ดู account details
  def show?
    member?
  end

  # แก้ไข account name, settings
  def update?
    admin?
  end

  # ลบ account — owner เท่านั้น
  def destroy?
    owner?
  end

  # จัดการ billing
  def manage_billing?
    owner?
  end

  # ดู members
  def members?
    member?
  end

  # Invite members ใหม่
  def invite?
    admin?
  end

  # Remove members
  def remove_member?
    admin?
  end

  # เปลี่ยน roles
  def change_roles?
    owner?
  end
end
```

### MembershipPolicy

```ruby
# app/policies/membership_policy.rb
class MembershipPolicy < ApplicationPolicy
  def show?
    # เห็น membership ของตัวเองหรือถ้าเป็น admin
    record.user_id == user.id || admin?
  end

  def create?
    admin?
  end

  # Update role
  def update?
    owner? && record.user_id != user.id  # owner ไม่เปลี่ยน role ตัวเอง
  end

  # ลบสมาชิก
  def destroy?
    # ลบตัวเองได้ หรือ admin ลบ member ได้ หรือ owner ลบทุกคนได้
    return true if record.user_id == user.id
    return false unless belongs_to_current_account?

    if record.owner?
      owner? && record.removable?
    elsif record.admin?
      owner?
    else
      admin?
    end
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      scope.where(account: account)
    end
  end
end
```

### ProjectPolicy สำหรับ Multi-tenant

```ruby
# app/policies/project_policy.rb
class ProjectPolicy < ApplicationPolicy
  def index?
    member?
  end

  def show?
    member? && belongs_to_current_account?
  end

  def create?
    member?
  end

  def update?
    # member เจ้าของ project หรือ admin
    belongs_to_current_account? && (record_owner? || admin?)
  end

  def destroy?
    belongs_to_current_account? && admin?
  end

  def archive?
    update?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      # ActsAsTenant handles tenant scoping,
      # แต่เราเพิ่ม filter ตาม visibility ด้วย
      scope.all
    end
  end

  private

  def record_owner?
    record.created_by_id == user.id
  end
end
```

### ใช้ Pundit ใน Controllers

```ruby
# app/controllers/memberships_controller.rb
class MembershipsController < ApplicationController
  before_action :set_membership, only: [:show, :update, :destroy]

  def index
    @memberships = policy_scope(Membership).includes(:user).order("users.name")
    authorize Membership
  end

  def update
    authorize @membership
    if @membership.update(membership_params)
      redirect_to settings_members_path, notice: "อัปเดต role สำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    authorize @membership
    unless @membership.removable?
      redirect_to settings_members_path, alert: "ไม่สามารถลบ owner คนสุดท้ายได้"
      return
    end

    @membership.destroy
    redirect_to settings_members_path, notice: "ลบสมาชิกออกแล้ว"
  end

  private

  def set_membership
    @membership = Membership.find(params[:id])
  end

  def membership_params
    params.require(:membership).permit(:role)
  end
end
```

---

## Step 933: Invite Flow — Invitation Model

ระบบ Invitation ที่ปลอดภัยด้วย secure random token, expiration และ role assignment

### Invitation Model

```bash
rails generate model Invitation \
  account:references \
  inviter:references \
  email:string \
  role:string \
  token:string \
  expires_at:datetime \
  accepted_at:datetime \
  revoked_at:datetime
```

```ruby
# app/models/invitation.rb
class Invitation < ApplicationRecord
  include TenantScoped

  belongs_to :account
  belongs_to :inviter, class_name: "User"
  has_one :membership

  TOKEN_EXPIRY = 7.days

  validates :email, presence: true,
    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :role, inclusion: { in: Membership::ROLES }
  validates :token, presence: true, uniqueness: true

  # ป้องกันการ invite email เดิมซ้ำ (ถ้า pending อยู่)
  validate :no_duplicate_pending_invitation
  validate :invitee_not_already_member

  before_validation :generate_token, on: :create
  before_create :set_expiry

  # Scopes
  scope :pending, -> { where(accepted_at: nil, revoked_at: nil).where("expires_at > ?", Time.current) }
  scope :accepted, -> { where.not(accepted_at: nil) }
  scope :expired, -> { where("expires_at <= ?", Time.current).where(accepted_at: nil) }
  scope :recent, -> { order(created_at: :desc) }

  def pending?
    accepted_at.nil? && revoked_at.nil? && expires_at > Time.current
  end

  def accepted?
    accepted_at.present?
  end

  def expired?
    expires_at <= Time.current && !accepted?
  end

  def revoked?
    revoked_at.present?
  end

  def accept!(user)
    return false unless pending?

    ActiveRecord::Base.transaction do
      membership = Membership.create!(
        user: user,
        account: account,
        role: role,
        invited_by: inviter,
        joined_at: Time.current
      )
      update!(accepted_at: Time.current)
      membership
    end
  end

  def revoke!
    update!(revoked_at: Time.current)
  end

  def invitation_url
    Rails.application.routes.url_helpers.accept_invitation_url(
      token: token,
      host: Rails.application.config.app_host
    )
  end

  private

  def generate_token
    self.token = SecureRandom.urlsafe_base64(32)
  end

  def set_expiry
    self.expires_at ||= TOKEN_EXPIRY.from_now
  end

  def no_duplicate_pending_invitation
    existing = Invitation.pending.where(account: account, email: email)
    existing = existing.where.not(id: id) if persisted?

    if existing.exists?
      errors.add(:email, "มี invitation ที่ยังรออยู่สำหรับ email นี้แล้ว")
    end
  end

  def invitee_not_already_member
    user = User.find_by(email: email)
    if user && account.memberships.exists?(user: user)
      errors.add(:email, "ผู้ใช้นี้เป็นสมาชิกของ account อยู่แล้ว")
    end
  end
end
```

### InvitationsController

```ruby
# app/controllers/invitations_controller.rb
class InvitationsController < ApplicationController
  skip_before_action :set_current_tenant, only: [:show, :accept]
  skip_before_action :authenticate_user!, only: [:show, :accept]

  def new
    authorize :invitation, :create?
    @invitation = Invitation.new
  end

  def create
    authorize :invitation, :create?

    @invitation = Invitation.new(invitation_params)
    @invitation.account = Current.account
    @invitation.inviter = current_user

    if @invitation.save
      InvitationMailer.invite(@invitation).deliver_later
      redirect_to settings_members_path,
        notice: "ส่ง Invitation ให้ #{@invitation.email} แล้ว"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def show
    @invitation = Invitation.find_by!(token: params[:token])

    if @invitation.expired?
      render :expired and return
    end

    if @invitation.accepted?
      redirect_to root_path, notice: "Invitation นี้ถูก accept ไปแล้ว"
      return
    end

    @account = @invitation.account
  rescue ActiveRecord::RecordNotFound
    render :not_found
  end

  def accept
    @invitation = Invitation.find_by!(token: params[:token])

    unless @invitation.pending?
      redirect_to root_path, alert: "Invitation นี้ไม่สามารถใช้งานได้"
      return
    end

    if current_user
      handle_accept_for_logged_in_user
    else
      # เก็บ token ใน session แล้วให้ login/signup ก่อน
      session[:pending_invitation_token] = params[:token]
      redirect_to new_user_registration_path,
        notice: "กรุณาสมัครสมาชิกหรือ login เพื่อรับ invitation"
    end
  rescue ActiveRecord::RecordNotFound
    render :not_found
  end

  def destroy
    @invitation = Invitation.find(params[:id])
    authorize @invitation, :revoke?

    @invitation.revoke!
    redirect_to settings_members_path, notice: "ยกเลิก Invitation แล้ว"
  end

  private

  def handle_accept_for_logged_in_user
    membership = @invitation.accept!(current_user)

    if membership
      AuditLog.record(
        account: @invitation.account,
        user: current_user,
        action: "invitation_accepted",
        metadata: { invited_by: @invitation.inviter_id, role: @invitation.role }
      )
      redirect_to account_dashboard_url(@invitation.account),
        notice: "ยินดีต้อนรับสู่ #{@invitation.account.name}!"
    else
      redirect_to root_path, alert: "ไม่สามารถ join account ได้"
    end
  end

  def invitation_params
    params.require(:invitation).permit(:email, :role)
  end
end
```

### InvitationMailer

```ruby
# app/mailers/invitation_mailer.rb
class InvitationMailer < ApplicationMailer
  def invite(invitation)
    @invitation = invitation
    @account = invitation.account
    @inviter = invitation.inviter
    @accept_url = accept_invitation_url(token: invitation.token)
    @expires_at = invitation.expires_at

    mail(
      to: invitation.email,
      subject: "#{@inviter.display_name} ได้ invite คุณเข้าร่วม #{@account.name} บน ProjectFlow"
    )
  end
end
```

```erb
<!-- app/views/invitation_mailer/invite.html.erb -->
<!DOCTYPE html>
<html>
<body>
  <div style="max-width: 600px; margin: 0 auto; font-family: sans-serif;">
    <h1>คุณได้รับ Invitation!</h1>

    <p>
      <strong><%= @inviter.display_name %></strong> ได้ invite คุณเข้าร่วม
      <strong><%= @account.name %></strong> ในฐานะ <strong><%= @invitation.role %></strong>
    </p>

    <a href="<%= @accept_url %>"
       style="display: inline-block; padding: 12px 24px; background: #2563eb;
              color: white; text-decoration: none; border-radius: 6px;">
      รับ Invitation
    </a>

    <p style="color: #6b7280; font-size: 0.875rem;">
      Invitation นี้หมดอายุในวันที่ <%= @expires_at.strftime("%d %B %Y") %>
    </p>
  </div>
</body>
</html>
```

---

## Step 934: Accept Invitation

ขั้นตอนการ Accept Invitation สำหรับทั้ง user ที่มีบัญชีอยู่แล้วและ user ใหม่

### Post-Registration Hook

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < Devise::RegistrationsController
  after_action :accept_pending_invitation, only: :create

  private

  def accept_pending_invitation
    return unless resource.persisted?  # การสมัครสำเร็จ
    return unless session[:pending_invitation_token].present?

    token = session.delete(:pending_invitation_token)
    invitation = Invitation.find_by(token: token)

    return unless invitation&.pending?

    membership = invitation.accept!(resource)

    if membership
      flash[:notice] = "ยินดีต้อนรับ! คุณเข้าร่วม #{invitation.account.name} แล้ว"
      # Store account เพื่อ redirect หลัง confirm email
      session[:joined_account_id] = invitation.account_id
    end
  end
end
```

### Post-Login Hook สำหรับ Pending Invitation

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < Devise::SessionsController
  after_action :process_pending_invitation, only: :create

  private

  def process_pending_invitation
    return unless resource.present? && session[:pending_invitation_token].present?

    token = session.delete(:pending_invitation_token)
    invitation = Invitation.find_by(token: token)

    return unless invitation&.pending?

    # ตรวจสอบว่า email ตรงกัน
    unless resource.email.casecmp?(invitation.email)
      flash[:alert] = "Invitation นี้ถูกส่งให้ #{invitation.email} ไม่ใช่ #{resource.email}"
      return
    end

    membership = invitation.accept!(resource)

    if membership
      AuditLog.record(
        account: invitation.account,
        user: resource,
        action: "invitation_accepted",
        metadata: { role: invitation.role }
      )
      flash[:notice] = "คุณเข้าร่วม #{invitation.account.name} แล้ว!"
    end
  end
end
```

---

## Step 935: Organization Settings Page

หน้า Organization Settings สำหรับจัดการ members, roles และ account configuration

### Routes

```ruby
# config/routes.rb
namespace :settings do
  resource :account, only: [:show, :update, :destroy]
  resources :members, only: [:index, :update, :destroy]
  resources :invitations, only: [:new, :create, :destroy]
  resource :billing, only: [:show]
end
```

### Settings::MembersController

```ruby
# app/controllers/settings/members_controller.rb
class Settings::MembersController < ApplicationController
  before_action :require_admin!

  def index
    @memberships = policy_scope(Membership)
      .includes(:user)
      .order(:role, "users.name")
    @invitations = Invitation.pending.recent.includes(:inviter)

    authorize Membership
  end

  def update
    @membership = Membership.find(params[:id])
    authorize @membership

    old_role = @membership.role

    if @membership.update(role: params[:membership][:role])
      AuditLog.record(
        account: Current.account,
        user: current_user,
        action: "role_changed",
        metadata: {
          target_user_id: @membership.user_id,
          old_role: old_role,
          new_role: @membership.role
        }
      )
      redirect_to settings_members_path,
        notice: "เปลี่ยน role ของ #{@membership.user.display_name} เป็น #{@membership.role} แล้ว"
    else
      render :index, status: :unprocessable_entity
    end
  end

  def destroy
    @membership = Membership.find(params[:id])
    authorize @membership

    unless @membership.removable?
      redirect_to settings_members_path,
        alert: "ไม่สามารถลบ owner คนสุดท้ายออกได้"
      return
    end

    user_name = @membership.user.display_name

    AuditLog.record(
      account: Current.account,
      user: current_user,
      action: "member_removed",
      metadata: { removed_user_id: @membership.user_id, removed_role: @membership.role }
    )

    @membership.destroy
    redirect_to settings_members_path, notice: "ลบ #{user_name} ออกจาก team แล้ว"
  end

  private

  def require_admin!
    unless Current.admin?
      redirect_to root_path, alert: "ต้องการสิทธิ์ Admin"
    end
  end
end
```

### Members Index View

```erb
<!-- app/views/settings/members/index.html.erb -->
<div class="max-w-4xl mx-auto px-4 py-8">
  <div class="flex justify-between items-center mb-6">
    <h1 class="text-2xl font-bold">สมาชิกใน Team</h1>
    <% if policy(:invitation).create? %>
      <%= link_to "Invite สมาชิก", new_settings_invitation_path,
        class: "px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700" %>
    <% end %>
  </div>

  <!-- Active Members -->
  <div class="bg-white rounded-lg shadow overflow-hidden mb-6">
    <table class="w-full">
      <thead class="bg-gray-50">
        <tr>
          <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">ชื่อ</th>
          <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Role</th>
          <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">เข้าร่วมเมื่อ</th>
          <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Active ล่าสุด</th>
          <th></th>
        </tr>
      </thead>
      <tbody class="divide-y divide-gray-200">
        <% @memberships.each do |membership| %>
          <tr>
            <td class="px-6 py-4">
              <div class="flex items-center gap-3">
                <div class="w-8 h-8 rounded-full bg-blue-100 flex items-center justify-center text-sm font-medium text-blue-600">
                  <%= membership.user.initials %>
                </div>
                <div>
                  <p class="font-medium"><%= membership.user.display_name %></p>
                  <p class="text-sm text-gray-500"><%= membership.user.email %></p>
                </div>
              </div>
            </td>
            <td class="px-6 py-4">
              <% if policy(membership).update? %>
                <%= form_with url: settings_member_path(membership), method: :patch, local: true do |f| %>
                  <%= f.select :role,
                    Membership::ROLES.map { |r| [r.humanize, r] },
                    { selected: membership.role },
                    { class: "text-sm border rounded px-2 py-1", onchange: "this.form.submit()" } %>
                <% end %>
              <% else %>
                <span class="px-2 py-1 bg-gray-100 text-gray-700 rounded text-sm">
                  <%= membership.display_role %>
                </span>
              <% end %>
            </td>
            <td class="px-6 py-4 text-sm text-gray-500">
              <%= membership.joined_at&.strftime("%d %b %Y") || "-" %>
            </td>
            <td class="px-6 py-4 text-sm text-gray-500">
              <%= membership.last_active_at ? time_ago_in_words(membership.last_active_at) + " ago" : "ยังไม่เคย" %>
            </td>
            <td class="px-6 py-4 text-right">
              <% if policy(membership).destroy? %>
                <%= button_to "ลบ",
                  settings_member_path(membership),
                  method: :delete,
                  data: { confirm: "ต้องการลบ #{membership.user.display_name} ออกใช่ไหม?" },
                  class: "text-sm text-red-600 hover:text-red-800" %>
              <% end %>
            </td>
          </tr>
        <% end %>
      </tbody>
    </table>
  </div>

  <!-- Pending Invitations -->
  <% if @invitations.any? %>
    <h2 class="text-lg font-semibold mb-4">Invitations ที่รออยู่</h2>
    <div class="bg-white rounded-lg shadow overflow-hidden">
      <% @invitations.each do |invitation| %>
        <div class="flex items-center justify-between px-6 py-4 border-b last:border-0">
          <div>
            <p class="font-medium"><%= invitation.email %></p>
            <p class="text-sm text-gray-500">
              Invited โดย <%= invitation.inviter.display_name %> ·
              หมดอายุใน <%= time_ago_in_words(invitation.expires_at) %>
            </p>
          </div>
          <span class="px-2 py-1 bg-yellow-100 text-yellow-800 rounded text-sm">
            <%= invitation.role %>
          </span>
          <%= button_to "ยกเลิก",
            settings_invitation_path(invitation),
            method: :delete,
            class: "text-sm text-red-600 hover:text-red-800 ml-4" %>
        </div>
      <% end %>
    </div>
  <% end %>
</div>
```

---

## Step 936: Audit Log

Audit Log ติดตาม actions ที่สำคัญเพื่อ security, compliance และ debugging

### Audit Log Model

```bash
rails generate model AuditLog \
  account:references \
  user:references \
  action:string \
  resource_type:string \
  resource_id:bigint \
  metadata:jsonb \
  ip_address:string \
  user_agent:string
```

```ruby
# app/models/audit_log.rb
class AuditLog < ApplicationRecord
  belongs_to :account
  belongs_to :user, optional: true  # บาง events อาจเกิดโดย system

  # Actions ที่สำคัญ
  SENSITIVE_ACTIONS = %w[
    invitation_sent
    invitation_accepted
    role_changed
    member_removed
    billing_updated
    plan_upgraded
    plan_downgraded
    subscription_canceled
    api_key_created
    api_key_revoked
    account_settings_changed
    owner_transferred
  ].freeze

  validates :action, presence: true

  # Scopes
  scope :recent, -> { order(created_at: :desc) }
  scope :for_action, ->(action) { where(action: action) }
  scope :for_resource, ->(type, id) { where(resource_type: type, resource_id: id) }
  scope :sensitive, -> { where(action: SENSITIVE_ACTIONS) }

  # Class method สำหรับ record log ง่ายๆ
  def self.record(account:, action:, user: nil, resource: nil, metadata: {}, request: nil)
    attrs = {
      account: account,
      user: user || Current.user,
      action: action.to_s,
      metadata: metadata
    }

    if resource
      attrs[:resource_type] = resource.class.name
      attrs[:resource_id] = resource.id
    end

    if request
      attrs[:ip_address] = request.remote_ip
      attrs[:user_agent] = request.user_agent
    end

    create!(attrs)
  rescue => e
    # Audit log ไม่ควร crash application
    Rails.logger.error "AuditLog.record failed: #{e.message}"
    nil
  end

  def display_action
    I18n.t("audit_actions.#{action}", default: action.humanize)
  end

  def sensitive?
    SENSITIVE_ACTIONS.include?(action)
  end
end
```

### AuditLog Concern สำหรับ Controllers

```ruby
# app/controllers/concerns/auditable.rb
module Auditable
  extend ActiveSupport::Concern

  private

  def audit_log(action, resource: nil, metadata: {})
    AuditLog.record(
      account: Current.account,
      user: current_user,
      action: action,
      resource: resource,
      metadata: metadata,
      request: request
    )
  end
end
```

### Audit Log View

```ruby
# app/controllers/settings/audit_logs_controller.rb
class Settings::AuditLogsController < ApplicationController
  before_action :require_admin!

  def index
    @audit_logs = AuditLog
      .where(account: Current.account)
      .includes(:user)
      .recent
      .limit(100)

    # Filter
    @audit_logs = @audit_logs.for_action(params[:action]) if params[:action].present?
    @audit_logs = @audit_logs.where(user_id: params[:user_id]) if params[:user_id].present?
    @audit_logs = @audit_logs.sensitive if params[:sensitive] == "true"

    authorize @audit_logs
  end
end
```

---

## Step 937: Account Switching

User ที่เป็นสมาชิกของหลาย accounts ต้องสามารถ switch ระหว่าง accounts ได้ง่าย

### Account Switcher Controller

```ruby
# app/controllers/account_switches_controller.rb
class AccountSwitchesController < ApplicationController
  skip_before_action :set_current_tenant

  def create
    account = current_user.accounts.find(params[:account_id])

    # บันทึก account ที่เลือกไว้ใน session
    session[:current_account_id] = account.id

    redirect_to account_dashboard_url(subdomain: account.subdomain),
      notice: "เปลี่ยนไปยัง #{account.name} แล้ว"

  rescue ActiveRecord::RecordNotFound
    redirect_to root_path, alert: "ไม่พบ account หรือคุณไม่ได้เป็นสมาชิก"
  end
end
```

### Account Switcher UI Component

```erb
<!-- app/views/shared/_account_switcher.html.erb -->
<div class="relative" data-controller="dropdown">
  <button
    data-action="click->dropdown#toggle"
    class="flex items-center gap-2 px-3 py-2 rounded-lg hover:bg-gray-100"
  >
    <div class="w-6 h-6 rounded bg-blue-600 text-white text-xs flex items-center justify-center font-bold">
      <%= Current.account.name.first.upcase %>
    </div>
    <span class="font-medium text-sm"><%= Current.account.name %></span>
    <svg class="w-4 h-4 text-gray-400"><!-- chevron --></svg>
  </button>

  <div
    data-dropdown-target="menu"
    class="hidden absolute left-0 top-full mt-1 w-64 bg-white rounded-lg shadow-lg border z-50"
  >
    <div class="p-2">
      <p class="text-xs text-gray-500 px-2 py-1 uppercase font-medium">Accounts ของคุณ</p>

      <% current_user.accounts.includes(:memberships).each do |account| %>
        <% membership = account.memberships.find_by(user: current_user) %>
        <div class="flex items-center gap-2 px-2 py-2 rounded hover:bg-gray-50 <%= 'bg-blue-50' if account == Current.account %>">
          <div class="w-8 h-8 rounded bg-blue-100 text-blue-600 text-sm flex items-center justify-center font-bold">
            <%= account.name.first.upcase %>
          </div>
          <div class="flex-1 min-w-0">
            <p class="text-sm font-medium truncate"><%= account.name %></p>
            <p class="text-xs text-gray-500"><%= membership&.role&.humanize %></p>
          </div>

          <% if account != Current.account %>
            <%= button_to account_switch_path(account_id: account.id),
              method: :post,
              class: "text-xs text-blue-600 hover:underline" do %>
              Switch
            <% end %>
          <% else %>
            <svg class="w-4 h-4 text-blue-600"><!-- checkmark --></svg>
          <% end %>
        </div>
      <% end %>

      <hr class="my-2">
      <%= link_to "สร้าง Account ใหม่", new_account_path,
        class: "flex items-center gap-2 px-2 py-2 text-sm text-gray-600 hover:bg-gray-50 rounded" %>
    </div>
  </div>
</div>
```

---

## Step 938: API Access สำหรับ SaaS

API Key per Account พร้อม Rate Limiting ตาม Plan ช่วยให้ลูกค้าสามารถ integrate กับ ProjectFlow ผ่าน API ได้

### ApiKey Model

```bash
rails generate model ApiKey \
  account:references \
  created_by:references \
  name:string \
  key_digest:string \
  last_used_at:datetime \
  revoked_at:datetime \
  expires_at:datetime
```

```ruby
# app/models/api_key.rb
class ApiKey < ApplicationRecord
  include TenantScoped

  belongs_to :account
  belongs_to :created_by, class_name: "User"

  KEY_PREFIX = "pf_"  # ProjectFlow prefix
  KEY_LENGTH = 32

  validates :name, presence: true, length: { maximum: 100 }
  validates :key_digest, presence: true

  scope :active, -> { where(revoked_at: nil).where("expires_at IS NULL OR expires_at > ?", Time.current) }
  scope :recent, -> { order(created_at: :desc) }

  # Generate key ใหม่ (ส่งคืน plain key ครั้งเดียว)
  def self.generate_for(account:, created_by:, name:)
    plain_key = "#{KEY_PREFIX}#{SecureRandom.hex(KEY_LENGTH)}"
    key = new(
      account: account,
      created_by: created_by,
      name: name,
      key_digest: digest(plain_key)
    )
    key.save!
    [key, plain_key]  # ส่งคืน plain key ครั้งเดียวเท่านั้น
  end

  # ค้นหา key จาก plain key
  def self.authenticate(plain_key)
    return nil unless plain_key&.start_with?(KEY_PREFIX)
    key_digest = digest(plain_key)
    active.find_by(key_digest: key_digest)
  end

  def self.digest(plain_key)
    Digest::SHA256.hexdigest(plain_key)
  end

  def active?
    revoked_at.nil? && (expires_at.nil? || expires_at > Time.current)
  end

  def revoke!
    update!(revoked_at: Time.current)
  end

  def touch_usage!
    update_column(:last_used_at, Time.current)
  end
end
```

### API Authentication

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      before_action :authenticate_api_key!
      before_action :check_api_rate_limit!
      before_action :check_api_feature!

      private

      def authenticate_api_key!
        api_key_value = request.headers["Authorization"]&.sub(/^Bearer /, "")

        unless api_key_value.present?
          render_unauthorized("API key required")
          return
        end

        @api_key = ApiKey.authenticate(api_key_value)

        unless @api_key
          render_unauthorized("Invalid or expired API key")
          return
        end

        @api_key.touch_usage!
        Current.account = @api_key.account
      end

      def check_api_feature!
        unless feature_available?(:api_access)
          render json: {
            error: "API access requires Pro or Enterprise plan",
            upgrade_url: billing_url
          }, status: :payment_required
        end
      end

      def check_api_rate_limit!
        limit = PlanLimits.limit_for(Current.account.plan, :api_calls_per_month)
        return if limit == -1  # unlimited

        key = "api_rate_limit:#{Current.account.id}:#{Date.today.strftime('%Y-%m')}"
        count = Rails.cache.increment(key, 1, expires_in: 31.days)

        if count > limit
          response.set_header("X-RateLimit-Limit", limit.to_s)
          response.set_header("X-RateLimit-Remaining", "0")
          render json: {
            error: "Rate limit exceeded",
            limit: limit,
            resets_at: Date.today.end_of_month.to_s
          }, status: :too_many_requests
        else
          response.set_header("X-RateLimit-Limit", limit.to_s)
          response.set_header("X-RateLimit-Remaining", [limit - count, 0].max.to_s)
        end
      end

      def render_unauthorized(message)
        render json: { error: message }, status: :unauthorized
      end

      def feature_available?(feature)
        PlanLimits.feature_available?(Current.account.plan, feature.to_s)
      end
    end
  end
end
```

---

## Step 939: Onboarding Flow

Welcome Wizard ช่วย user ใหม่ setup account ได้อย่างรวดเร็วและสมบูรณ์

### Onboarding State Machine

```ruby
# app/models/concerns/onboardable.rb
module Onboardable
  extend ActiveSupport::Concern

  ONBOARDING_STEPS = %w[
    profile_setup
    invite_team
    create_first_project
    connect_integration
    completed
  ].freeze

  included do
    store_accessor :metadata, :onboarding_step, :onboarding_completed_at
  end

  def onboarding_completed?
    onboarding_step == "completed"
  end

  def current_onboarding_step
    onboarding_step || ONBOARDING_STEPS.first
  end

  def complete_onboarding_step!(step)
    current_index = ONBOARDING_STEPS.index(step.to_s)
    return false unless current_index

    next_step = ONBOARDING_STEPS[current_index + 1] || "completed"
    update!(
      onboarding_step: next_step,
      onboarding_completed_at: (next_step == "completed" ? Time.current : nil)
    )
  end

  def onboarding_progress
    return 100 if onboarding_completed?
    current_index = ONBOARDING_STEPS.index(current_onboarding_step) || 0
    ((current_index.to_f / (ONBOARDING_STEPS.count - 1)) * 100).round
  end
end
```

### Onboarding Controller

```ruby
# app/controllers/onboarding_controller.rb
class OnboardingController < ApplicationController
  before_action :redirect_if_completed

  STEPS = Account::ONBOARDING_STEPS.freeze

  def show
    @step = current_step
    @account = Current.account
    render "onboarding/steps/#{@step}"
  end

  def update
    @step = current_step
    @account = Current.account

    result = process_step(@step)

    if result[:success]
      Current.account.complete_onboarding_step!(@step)

      if Current.account.onboarding_completed?
        redirect_to dashboard_path, notice: "ยินดีต้อนรับ! เริ่มใช้งาน ProjectFlow ได้เลย 🎉"
      else
        redirect_to onboarding_path, notice: result[:message]
      end
    else
      flash.now[:alert] = result[:error]
      render "onboarding/steps/#{@step}", status: :unprocessable_entity
    end
  end

  private

  def current_step
    Current.account.current_onboarding_step
  end

  def process_step(step)
    case step
    when "profile_setup"
      process_profile_setup
    when "invite_team"
      process_invite_team
    when "create_first_project"
      process_first_project
    when "connect_integration"
      { success: true, message: "ข้ามขั้นตอนนี้แล้ว" }
    end
  end

  def process_profile_setup
    if Current.account.update(account_params)
      current_user.update(user_params)
      { success: true, message: "ตั้งค่า Profile เสร็จแล้ว!" }
    else
      { success: false, error: Current.account.errors.full_messages.join(", ") }
    end
  end

  def process_invite_team
    emails = params[:emails].to_s.split(/[\s,]+/).map(&:strip).reject(&:blank?)

    if emails.empty?
      return { success: true, message: "ข้ามขั้นตอนนี้แล้ว" }
    end

    emails.each do |email|
      invitation = Invitation.new(
        account: Current.account,
        inviter: current_user,
        email: email,
        role: "member"
      )
      invitation.save && InvitationMailer.invite(invitation).deliver_later
    end

    { success: true, message: "ส่ง invitation ให้ #{emails.count} คนแล้ว!" }
  end

  def process_first_project
    project = Project.new(
      name: params[:project_name],
      account: Current.account
    )

    if project.save
      { success: true, message: "สร้าง Project แรกแล้ว!" }
    else
      { success: false, error: project.errors.full_messages.join(", ") }
    end
  end

  def redirect_if_completed
    if Current.account.onboarding_completed?
      redirect_to dashboard_path
    end
  end

  def account_params
    params.require(:account).permit(:name, :description)
  end

  def user_params
    params.require(:user).permit(:name, :avatar)
  end
end
```

---

## Step 940: SaaS Metrics Dashboard

SaaS Metrics Dashboard แสดง MRR, Churn Rate และ Active Users ด้วย Query Objects และ Charts

### Query Objects สำหรับ Metrics

```ruby
# app/queries/saas_metrics_query.rb
class SaasMetricsQuery
  def initialize(date_range: 12.months.ago..Time.current)
    @date_range = date_range
  end

  # Monthly Recurring Revenue
  def mrr
    active_subscriptions = Subscription
      .active
      .joins(:account)
      .where(created_at: @date_range)

    active_subscriptions.sum { |sub| plan_monthly_revenue(sub.plan) }
  end

  # MRR by month
  def mrr_by_month
    months = generate_months
    months.map do |month|
      subs = subscriptions_active_in_month(month)
      {
        month: month.strftime("%b %Y"),
        mrr: subs.sum { |s| plan_monthly_revenue(s.plan) }
      }
    end
  end

  # Churn Rate (monthly)
  def churn_rate_by_month
    months = generate_months
    months.map do |month|
      start_count = subscriptions_active_at(month.beginning_of_month).count
      churned = churned_in_month(month).count
      rate = start_count > 0 ? (churned.to_f / start_count * 100).round(2) : 0
      { month: month.strftime("%b %Y"), churn_rate: rate }
    end
  end

  # Active Users (ใช้งานในช่วง 30 วันล่าสุด)
  def active_users_count
    Membership
      .joins(:user)
      .where(last_active_at: 30.days.ago..)
      .select("DISTINCT user_id")
      .count
  end

  # New Signups by month
  def new_accounts_by_month
    Account
      .where(created_at: @date_range)
      .group("DATE_TRUNC('month', created_at)")
      .count
      .map { |month, count| { month: month.strftime("%b %Y"), count: count } }
  end

  # Plan Distribution
  def plan_distribution
    Account
      .joins(:subscription)
      .where(subscriptions: { status: %w[active trialing] })
      .group("subscriptions.plan")
      .count
  end

  private

  def plan_monthly_revenue(plan)
    case plan
    when "pro" then 29
    when "enterprise" then 99
    else 0
    end
  end

  def generate_months
    start_date = 12.months.ago.beginning_of_month
    (0..11).map { |i| start_date + i.months }
  end

  def subscriptions_active_in_month(month)
    Subscription
      .active
      .where("created_at <= ? AND (canceled_at IS NULL OR canceled_at >= ?)",
             month.end_of_month, month.beginning_of_month)
  end

  def subscriptions_active_at(date)
    Subscription
      .where("created_at <= ? AND (canceled_at IS NULL OR canceled_at >= ?)", date, date)
  end

  def churned_in_month(month)
    Subscription
      .where(status: "canceled")
      .where(canceled_at: month.beginning_of_month..month.end_of_month)
  end
end
```

### Metrics Dashboard Controller

```ruby
# app/controllers/admin/metrics_controller.rb
class Admin::MetricsController < ApplicationController
  before_action :require_super_admin!
  skip_before_action :set_current_tenant

  def index
    @metrics = SaasMetricsQuery.new

    @stats = {
      mrr: @metrics.mrr,
      active_users: @metrics.active_users_count,
      total_accounts: Account.count,
      active_subscriptions: Subscription.active.count,
      churn_rate: calculate_current_churn_rate
    }

    @mrr_chart_data = @metrics.mrr_by_month
    @churn_chart_data = @metrics.churn_rate_by_month
    @new_accounts_data = @metrics.new_accounts_by_month
    @plan_distribution = @metrics.plan_distribution
  end

  private

  def require_super_admin!
    redirect_to root_path unless current_user&.super_admin?
  end

  def calculate_current_churn_rate
    data = @metrics.churn_rate_by_month
    data.last&.dig(:churn_rate) || 0
  end
end
```

### Metrics Dashboard View

```erb
<!-- app/views/admin/metrics/index.html.erb -->
<div class="max-w-7xl mx-auto px-4 py-8">
  <h1 class="text-2xl font-bold mb-8">SaaS Metrics Dashboard</h1>

  <!-- KPI Cards -->
  <div class="grid grid-cols-2 md:grid-cols-5 gap-4 mb-8">
    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">MRR</p>
      <p class="text-2xl font-bold text-green-600">$<%= number_with_delimiter(@stats[:mrr]) %></p>
    </div>

    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">Active Users (30d)</p>
      <p class="text-2xl font-bold"><%= number_with_delimiter(@stats[:active_users]) %></p>
    </div>

    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">Total Accounts</p>
      <p class="text-2xl font-bold"><%= number_with_delimiter(@stats[:total_accounts]) %></p>
    </div>

    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">Active Subscriptions</p>
      <p class="text-2xl font-bold"><%= number_with_delimiter(@stats[:active_subscriptions]) %></p>
    </div>

    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">Churn Rate (month)</p>
      <p class="text-2xl font-bold text-red-600"><%= @stats[:churn_rate] %>%</p>
    </div>
  </div>

  <!-- MRR Chart -->
  <div class="bg-white rounded-lg shadow p-6 mb-6">
    <h2 class="text-lg font-semibold mb-4">MRR Growth</h2>
    <div id="mrr-chart" data-chart-data="<%= @mrr_chart_data.to_json %>">
      <!-- Stimulus controller renders chart.js here -->
    </div>
  </div>

  <!-- Plan Distribution -->
  <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
    <div class="bg-white rounded-lg shadow p-6">
      <h2 class="text-lg font-semibold mb-4">Plan Distribution</h2>
      <% @plan_distribution.each do |plan, count| %>
        <div class="flex items-center justify-between py-2">
          <span class="capitalize font-medium"><%= plan %></span>
          <div class="flex items-center gap-2">
            <div class="w-32 bg-gray-200 rounded-full h-2">
              <div class="bg-blue-600 h-2 rounded-full"
                   style="width: <%= count.to_f / @stats[:total_accounts] * 100 %>%">
              </div>
            </div>
            <span class="text-sm text-gray-600"><%= count %></span>
          </div>
        </div>
      <% end %>
    </div>

    <div class="bg-white rounded-lg shadow p-6">
      <h2 class="text-lg font-semibold mb-4">Churn Rate (12 months)</h2>
      <div id="churn-chart" data-chart-data="<%= @churn_chart_data.to_json %>">
        <!-- Chart renders here -->
      </div>
    </div>
  </div>
</div>
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Implement Permission Matrix

สร้าง permission matrix ที่ละเอียดขึ้น โดยแต่ละ role มี permissions ที่กำหนดไว้ชัดเจน:

```ruby
# app/services/permission_matrix.rb
class PermissionMatrix
  PERMISSIONS = {
    owner: {
      billing: [:read, :update, :cancel],
      members: [:read, :invite, :update_role, :remove],
      account: [:read, :update, :delete],
      projects: [:read, :create, :update, :delete, :archive]
    },
    admin: {
      billing: [:read],
      members: [:read, :invite, :remove],  # ไม่เปลี่ยน owner role
      account: [:read, :update],
      projects: [:read, :create, :update, :delete, :archive]
    },
    member: {
      billing: [],
      members: [:read],
      account: [:read],
      projects: [:read, :create, :update]  # ลบได้เฉพาะ project ตัวเอง
    }
  }.freeze

  def self.can?(role, resource, action)
    PERMISSIONS.dig(role.to_sym, resource.to_sym)&.include?(action.to_sym) || false
  end
end
```

### แบบฝึกหัดที่ 2: Invitation Reminder Job

สร้าง background job ส่ง reminder สำหรับ invitations ที่ใกล้หมดอายุ:

```ruby
class InvitationReminderJob < ApplicationJob
  queue_as :mailers

  def perform
    # Invitations ที่จะหมดอายุใน 24 ชั่วโมง และยังไม่ได้รับ
    expiring_soon = Invitation
      .pending
      .where(expires_at: Time.current..24.hours.from_now)

    expiring_soon.find_each do |invitation|
      InvitationMailer.reminder(invitation).deliver_now
    end
  end
end
```

### แบบฝึกหัดที่ 3: Expand Metrics ด้วย Revenue per Plan

เพิ่ม breakdown ของ MRR ตาม plan:

```ruby
def mrr_by_plan
  Subscription
    .active
    .group(:plan)
    .sum { |subs| subs.map { |s| plan_monthly_revenue(s.plan) }.sum }
end
```

---

## สรุปสิ่งที่ได้เรียนรู้

ใน Part 094 นี้ เราได้เรียนรู้:

1. **Membership Join Table** — การออกแบบ join table ที่รองรับ roles (owner/admin/member) และ tracking activity ของ members

2. **Pundit Multi-tenant Policies** — การ implement Pundit authorization ที่ตระหนักถึง tenant context โดยส่ง `Current` object แทน user ธรรมดา

3. **Invitation Flow** — ระบบ invitation ที่ปลอดภัยด้วย secure token, expiration, email verification และ duplicate protection

4. **Accept Invitation** — การ handle invitation acceptance ทั้งสำหรับ user ที่มีบัญชีอยู่แล้วและ user ใหม่ผ่าน post-registration/login hooks

5. **Organization Settings** — หน้าจัดการสมาชิกที่ครบถ้วน ทั้ง list, update role, remove member และ pending invitations

6. **Audit Log** — การ record sensitive actions เพื่อ security และ compliance โดยไม่ crash application

7. **Account Switching** — UI ที่ช่วยให้ user ที่อยู่หลาย accounts สามารถ switch ได้อย่างรวดเร็ว

8. **API Keys** — การสร้างและ authenticate API keys ด้วย SHA256 digest พร้อม rate limiting ตาม plan

9. **Onboarding Flow** — Welcome wizard ที่ guide user ใหม่ผ่าน steps ที่จำเป็นด้วย state machine pattern

10. **SaaS Metrics** — Query objects สำหรับคำนวณ MRR, Churn Rate และ Active Users พร้อม dashboard ที่ visualize ข้อมูล

---

## Pattern สำคัญที่ควรจำ

```ruby
# 1. Pundit กับ multi-tenant ใช้ Current object
class ApplicationPolicy
  def initialize(current, record)  # current ไม่ใช่ user
    @current = current
  end
end

# 2. ActsAsTenant scopes ทุก query อัตโนมัติ
acts_as_tenant :account  # ใน model

# 3. Invitation accept ด้วย transaction
def accept!(user)
  ActiveRecord::Base.transaction do
    Membership.create!(user: user, account: account, role: role)
    update!(accepted_at: Time.current)
  end
end

# 4. API key ด้วย one-way digest
plain_key = "pf_#{SecureRandom.hex(32)}"
key_digest = Digest::SHA256.hexdigest(plain_key)

# 5. Rate limiting ด้วย Rails cache increment
count = Rails.cache.increment("rate:#{account.id}:#{Date.today}", 1, expires_in: 31.days)

# 6. MRR calculation ด้วย Query Object
class SaasMetricsQuery
  def mrr
    Subscription.active.sum { |sub| plan_monthly_revenue(sub.plan) }
  end
end
```

---

## Preview of Next Part

**Part 095: Capstone 2 — SaaS App: Real-time Features, Testing & Deployment**

ใน Part ถัดไป เราจะ complete SaaS application ด้วย:

- **Real-time Notifications** — ActionCable สำหรับ live updates และ in-app notifications
- **Turbo Streams** — Real-time UI updates โดยไม่ต้อง full page reload
- **Comprehensive Testing** — RSpec integration tests, Stripe mocking, feature specs
- **Security Hardening** — Rate limiting, CSRF protection, Content Security Policy
- **Performance Optimization** — N+1 query fixes, caching strategies, background jobs
- **Docker & CI/CD** — Containerization และ GitHub Actions deployment pipeline
- **Production Deployment** — Deploy to Heroku/Render พร้อม SSL, custom domain

เตรียมพร้อมสำหรับการ finalize และ deploy SaaS Application ของเราใน Part 095!
