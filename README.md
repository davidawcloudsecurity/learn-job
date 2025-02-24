# learn-job

To incorporate an if-else statement to set the RoleSessionName conditionally based on whether the string is not equal to 'quicksight', you can modify the _createUrl method as follows:

javascript
Copy code
async _createUrl () {
  const region = process.env.AWS_DEFAULT_REGION
  const isProject = process.env.IS_PROJECT === 'true'
  const federatedBaseUrl = `https://${region}.signin.aws.amazon.com/federation`
  let issuerUrl = process.env.PLATFORM_DOMAIN
  let signInToken

  if (isProject) {
    const project = await Project.findOne()

    if (!project) {
      throw new NotFoundError('Project not found')
    }

    issuerUrl = project.url

    if (this._sendToPlatform) {
      const platformBackendUrl = process.env.PLATFORM_BACKEND_URL
      const token = await getProjectToken()
      const platformResponse = await axios({
        method: 'GET',
        url: `${platformBackendUrl}/public-sts`,
        headers: {
          'x-auth-project-token': token
        },
        data: {
          policy: this._policy
        }
      })

      if (!platformResponse?.data) {
        throw new InternalServerError('Public STS Error')
      }

      logger.info(`platform response: ${JSON.stringify(platformResponse.data)}`)

      signInToken = platformResponse?.data?.body?.SigninToken

      const constructLoginUrlParams = {
        issuerUrl,
        signInToken,
        federatedBaseUrl
      }

      return this._constructLoginUrl(constructLoginUrlParams)
    }
  }

  let assumeRoleParams;

  // Check if the type is not 'quicksight'
  if (this._query.type !== 'quicksight') {
    assumeRoleParams = {
      RoleSessionName: 'genexis-session', // Set RoleSessionName to 'genexis-session'
      RoleArn: this._roleArn,
      Policy: this._policy
    }
  } else {
    // Default scenario
    assumeRoleParams = {
      RoleSessionName: this._res.locals.user.email,
      RoleArn: this._roleArn,
      Policy: this._policy
    }
  }

  const federationToken = await sts.assumeRole(assumeRoleParams).promise()

  const session = {
    sessionId: federationToken.Credentials.AccessKeyId,
    sessionKey: federationToken.Credentials.SecretAccessKey,
    sessionToken: federationToken.Credentials.SessionToken
  }

  const constructGetSignInUrlParams = {
    session,
    federatedBaseUrl
  }

  const getSignInUrl = this._constructGetSignInUrl(constructGetSignInUrlParams)
  const payload = { getSignInUrl }

  if (isProject) {
    const lambdaInvokerParams = {
      lambdaName: 'sso',
      functionName: 'get_federated_sign_in_token',
      payload
    }

    const platformBackendUrl = process.env.PLATFORM_BACKEND_URL
    const token = await getProjectToken()
    const platformResponse = await axios({
      method: 'GET',
      url: `${platformBackendUrl}/lambda/invoke`,
      headers: {
        'x-auth-project-token': token
      },
      data: lambdaInvokerParams
    })

    if (!platformResponse?.data) {
      throw new InternalServerError('Lambda Invoke Get Sign In Token Error')
    }

    logger.info(`platform response: ${JSON.stringify(platformResponse.data)}`)

    signInToken = platformResponse?.data?.body?.SigninToken
  } else {
    const getSignInData = await lambdaInvokerHandler('sso', 'get_federated_sign_in_token', payload)

    logger.info(`Lambda invoker STS response: ${JSON.stringify(getSignInData)}`)

    signInToken = getSignInData?.SigninToken
  }

  const constructLoginUrlParams = {
    issuerUrl,
    signInToken,
    federatedBaseUrl
  }

  return this._constructLoginUrl(constructLoginUrlParams)
}
Explanation:
Condition Check:

Within _createUrl(), before the assumeRoleParams declaration, check if this._query.type is not equal to 'quicksight'.
If true, set assumeRoleParams.RoleSessionName to 'genexis-session'.
If false (meaning this._query.type is 'quicksight'), use this._res.locals.user.email as RoleSessionName.
Usage:

This logic ensures that RoleSessionName is dynamically set based on whether the type is 'quicksight' or not.
It handles scenarios where different session names are required based on the application's context (
