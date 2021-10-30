# Conventtion
1.await in for loop is eslint code smell. https://eslint.org/docs/rules/no-await-in-loop
Rewrite it with await Promise.all(), thanks.
```js
for (const unitRate of unitRates) {
  await queryRunner.query(`UPDATE "unit" SET "rate" = ${unitRate.rate} where "value" = '${unitRate.value}';`)
}

await queryRunner.query(`
  UPDATE "unit" SET "rate" = '38.7146' where "value" = 'fluid ounce(UK)' and "symbol" = 'fl oz(UK)';
  UPDATE "unit" SET "rate" = '37.1954' where "value" = 'fluid ounce(UK)' and "symbol" = 'fl oz(US)';
  UPDATE "unit" SET "rate" = '1.19599' where "value" like '%square yard';
`)
```
```js
await Promise.all([
...unitRates.map(unitRate =>
  queryRunner.query(`UPDATE "unit" SET "rate" = ${unitRate.rate} where "value" = '${unitRate.value}';`),
),
queryRunner.query(`
  UPDATE "unit" SET "rate" = '38.7146' where "value" = 'fluid ounce(UK)' and "symbol" = 'fl oz(UK)';
  UPDATE "unit" SET "rate" = '37.1954' where "value" = 'fluid ounce(UK)' and "symbol" = 'fl oz(US)';
  UPDATE "unit" SET "rate" = '1.19599' where "value" like '%square yard';
`),
])
```
2. await in for loop is a eslint code smell. We can flatten report members into a 1-d array and call manager.remove()
```js
for (const report of reports) {
  await manager.remove(ReportMember, report.members)
}
      
```
```js
const removeMembers = reports.map(report => report.members).flat()
await manager.remove(ReportMember, removeMembers)
```

3.Do all the operations in transaction because four tables are affected
Add manager parameter that is a an optional EntityManager
```js
await Promise.all([
  this.deleteUnpublishedReports({ organizationUserId: corporateUserId }),
  this.organizationUserRepository.update(corporateUserId, { pendingRemovalAt: new Date() }),
  this.userRepository.changeUserStatus([corporateUser.userId], USERSTATUS.PENDING_REMOVAL),
])
void this.sendBackupReportEmail(corporation, portfolioManager, [corporateUser.user]).then()
```
```js
await this.connection.transaction(async manager => {
  const organizationUserRepository = manager.getCustomRepository(OrganizationUserRepository)
  const userRepository = manager.getCustomRepository(UserRepository)

  await this.deleteUnpublishedReports({ organizationUserId: corporateUserId }, manager)
  await organizationUserRepository.update(corporateUserId, { pendingRemovalAt: new Date() })
  await userRepository.changeUserStatus([corporateUser.userId], USERSTATUS.PENDING_REMOVAL)
})

await this.sendBackupReportEmail(corporation, portfolioManager, [corporateUser.user])
```
4. Our Diginex Sonarcube scanner will flag this line as major error because reportItem.answers is sorted in place
It means do not change input parameter?
```js
async formatDisCompareAnswers(
  reportItem: ReportItem,
  units: Unit[],
  baseUnit: Unit | undefined,
  currenciesRate: Record<string, number> | undefined,
): Promise<AnswerFormat[]> {
  reportItem.answers.sort((a, b) => a.answerNumber - b.answerNumber)

```
```js
const sortedAnswers = [...reportItem.answers].sort((a, b) => a.answerNumber - b.answerNumber)
for (const answer of sortedAnswers) {
    ... the rest remains unchanged ...
}
```
5.Our Diginex Sonarqube scanner will mark it as minor code smell when switch statement has less than 3 case statements.
Please kindly rewrite it with if-elseif-else statements.
```js
switch (answer.unit?.category?.name) {
  case UNIT_CATEGORY.CURRENCY:
    if (!currenciesRate) {
      throw new BadRequestException(REPORT_COMPARISION_ERROR.INVALID_UNIT)
    }
    rate = currenciesRate[baseUnit?.value || '']
    answerUnitRate = currenciesRate[answer.unit?.value || '']
    break
  case UNIT_CATEGORY.EMPLOYEE:
    rate = 1
    answerUnitRate = 1
    break
  default:
    let unit = units.find(unit => unit.id === baseUnit.id)
    rate = unit?.rate
    unit = units.find(unit => unit.id === answer.unit?.id)
    answerUnitRate = unit?.rate
```
```js
if (unitCategoryName === UNIT_CATEGORY.CURRENCY) {
  if (!currenciesRate) {
    throw new BadRequestException(REPORT_COMPARISON_ERROR.INVALID_UNIT)
  }
  rate = currenciesRate[baseUnit?.value || '']
  answerUnitRate = currenciesRate[answer.unit?.value || '']
} else if (unitCategoryName === UNIT_CATEGORY.EMPLOYEE) {
  rate = 1
  answerUnitRate = 1
} else {
  rate = units.find(unit => unit.id === baseUnit.id)?.rate
  answerUnitRate = units.find(unit => unit.id === answer.unit?.id)?.rate
}
```
6. Convert deletedAt column to timestamp with timezone.
By trial and error, need to pass { type: 'timestamp with time zone' } to @DeleteDateColumn to get it to work.
```js
 @Column({
  type: 'timestamp with time zone',
  nullable: true,
})
@DeleteDateColumn()
```
```js
@DeleteDateColumn({ type: 'timestamp with time zone' })
```
7. Too many parameters in the constructor and it is a major code smell in sonarqube.
I have been refactoring the code base to keep the parameters to 7 or less and then apply max-params eslint rule (https://eslint.org/docs/rules/max-params)
Please kindly refactor the code and move functions to new services.

8.Techvify is doing consultancy feature where consultant (that is a type of use) can exist in multiple organizations.  As a precaution, please pass organization id to function and filter report queries by organization id.
```js
return this.getPublishedReportsByCorpUser(userId)
```
```js
return this.getPublishedReportsByCorpUser(organization.id, userId)
```

9.toPromise is deprecated in nestJS 8. Please import lastValueFrom from rxjs and rewrite with lastValueFrom
```js
const promise = this.httpService.get(this.configService.getOrException<string>('CURRENCY_EXCHANGE_API')).toPromise()
```
```js
const promise = lastValueFrom(this.httpService.get(this.configService.getOrException<string>('CURRENCY_EXCHANGE_API')))
```
10.Resolved by Promise.all for independent async tasks to fail fast.
```js
const organizations = await this.organizationRepository.getCorporationsByTemplateId(templateId)
const countReport = await reportRepository.countPublishedReportsByTemplateId(templateId)
```
```js
const [organizations, countReport] = await Promise.all([
  organizationRepository.getCorporationsByTemplateId(templateId),
  reportRepository.countPublishedReportsByTemplateId(templateId),
])
```
11.All database operations should perform within a database transaction
```js
 await this.connection.transaction(async manager => {
  const organizationUserRepository = manager.getCustomRepository(OrganizationUserRepository)
  const organizationUsers = await organizationUserRepository.find({
    where: { pendingRemovalAt: LessThanOrEqual(lastBackupDate) },
    relations: ['user'],
  })
  await this.deleteUsersPaymentInfo(organizationUsers, manager)

  const userIds = organizationUsers.map(organizationUser => organizationUser.userId)
  await organizationUserRepository.update(
    { pendingRemovalAt: LessThanOrEqual(lastBackupDate) },
    { valid: false, pendingRemovalAt: null },
  ),
  await manager.getCustomRepository(UserRepository).changeUserStatus(userIds, USERSTATUS.INACTIVE)
})

```
12.for await loop should be avoided as much as possible and replaced with Promise.all or Promise.allSettled().
```js
for (const orgUser of orgUsers) {
  const organization = await this.organizationRepository.getOrganizationByUserId(orgUser.user.id)
  if (!organization) {
    continue
  }

  const userInCharge = await this.paymentPlatformService.getUserInCharge(organization)
  const isPaidPremiumUser = await this.paymentPlatformService.isPaidPremiumUser(orgUser.user, userInCharge)
  if (isPaidPremiumUser) {
    if (!organization.discountId) {
      await this.paymentService.cancelSubscription(orgUser.userId)
    }
    await this.deleteOrgPaymentInfo(organization.id)
```
```js
const promises = organizations.map(async organization => {
  const userInCharge = await this.paymentPlatformService.getUserInCharge(organization, manager)
  const orgUserInCharge = organization.organizationUsers.find(orgUser =>
    this.paymentPlatformService.isPaidPremiumUser(orgUser.user, userInCharge),
  )
  let updateOption: Partial<Organization> = {
    status: ORGANIZATION_STATUS.DELETED,
    pendingRemovalAt: null,
    organizationId: await this.corporationService.generateOrganizationId(),
  }

  if (orgUserInCharge) {
    if (!organization.discountId) {
      await this.paymentSubscriptionService.cancelSubscription(orgUserInCharge.userId, manager)
    }

    updateOption = {
      ...updateOption,
      paymentPlatform: undefined,
      customerId: undefined,
      subscription: undefined,
      billingContact: undefined,
      discountId: undefined,
    }
  }

  await organizationRepository.update(organization.id, updateOption)
})

await Promise.allSettled(promises)
```
13.Circular dependency is formed  because repository depends or organization moduke and vice versa.
Move CoporateUser interface to repository and corporation controllers/services can import the interface by
```js
import { CorporationUser } from '@/organization/corporation/interfaces'
```
```js
import { CorporationUser } from '@/repository'
```
14.Create  new copy of corporation in map
```js
return corporations.map((corporation: Corporation) => {
  corporation.isPaidPremiumUser = corporation.subscription?.isValid
```
```js
return corporations.map((corporation: Corporation) => 
  ({ ...corporation, isPaidPremiumUser: corporation.subscription?.isValid })
)
```
14. unlock account and send invite email are dependent tasks.
We can group  unlock account and send email in one promise and create promises for each otp in the array
```js
this.organizationUsersService.unlockAccounts(otps, USER_STATUS.PENDING, 0)
this.corporationEmailService.sendInvitedUsersEmail(otps, portfolioManager, corporation)
```
```js
const promises = otps.map(async otp => {
try {
  await this.organizationUsersService.unlockAccounts([otp], USER_STATUS.PENDING, 0),
  await this.corporationEmailService.sendInvitedUsersEmail([otp], portfolioManager, corporation)
} catch (err) {
  throw new InternalServerErrorException(err)
}
})

if (promises && promises.length > 0) {
await Promise.allSettled(promises)
}
```
15.Cool trick
```js
async deleteUnpublishedReports(
  params: {
    templateId?: string
    organizationId?: string
    organizationUserId?: string
  },
  manager?: EntityManager,
): Promise<void> {
  const transactionHandler = async (manager: EntityManager) => {
    const reportRepository = manager.getCustomRepository(ReportRepository)

    const reports = await reportRepository.getUnpublishedReportsByTemplate(params)
    if (reports.length) {
      const removeMembers = reports.map(report => report.members).flat()
      await manager.remove(ReportMember, removeMembers)
      await manager.delete(
        Report,
        reports.map(report => report.id),
      )
    }
  }

  if (manager) {
    await transactionHandler(manager)
  } else {
    await this.connection.transaction(transactionHandler)
  }
}
```


