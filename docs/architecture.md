# Architecture

## Source Context

- Public slug: `aws-network-network-design`
- Topic: `aws-architecture`

## Evidence Notes


# bin/tap.ts
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import 'source-map-support/register';
import { TapStack } from '../lib/tap-stack';

const app = new cdk.App();

// Get environment suffix from environment variable, command line context, or use default
const environmentSuffix =
  process.env.ENVIRONMENT_SUFFIX ||
  app.node.tryGetContext('environmentSuffix') ||
  'dev';

new TapStack(app, `TapStack${environmentSuffix}`, {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: 'us-east-1',
  },
  environmentSuffix: environmentSuffix,
});


# lib/tap-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export interface TapStackProps extends cdk.StackProps {
  environmentSuffix: string;
}

export class TapStack extends cdk.Stack {
  private getCidrRanges(environmentSuffix: string) {
    // Generate unique CIDR ranges based on environment to avoid conflicts
    let baseCidr = '10.0.0.0/16';
    let subnet1Cidr = '10.0.1.0/24';
    let subnet2Cidr = '10.0.2.0/24';

    if (environmentSuffix === 'staging') {
      baseCidr = '10.1.0.0/16';
      subnet1Cidr = '10.1.1.0/24';
      subnet2Cidr = '10.1.2.0/24';
    } else if (environmentSuffix === 'prod') {
      baseCidr = '10.2.0.0/16';
      subnet1Cidr = '10.2.1.0/24';
      subnet2Cidr = '10.2.2.0/24';
    }

    return {
      vpcCidr: baseCidr,
      subnet1Cidr,
      subnet2Cidr,
    };
  }

  constructor(scope: Construct, id: string, props: TapStackProps) {
    super(scope, id, props);

    const { environmentSuffix } = props;
    const cidrRanges = this.getCidrRanges(environmentSuffix);

    // Create VPC with specified CIDR block
    const vpc = new ec2.Vpc(this, 'VPC', {
      ipAddresses: ec2.IpAddresses

# test/tap-stack.int.test.ts
// Integration tests for VPC infrastructure
import { EC2Client } from '@aws-sdk/client-ec2';
import fs from 'fs';

// Mock outputs for different environments
const mockOutputsByEnvironment = {
  dev: {
    VpcId: 'vpc-dev1234567890abcdef0',
    VpcCidr: '10.0.0.0/16',
    PublicSubnet1Id: 'subnet-dev1234567890abcdef0',
    PublicSubnet2Id: 'subnet-dev1234567890abcdef1',
    PublicSubnet1Az: 'us-east-1a',
    PublicSubnet2Az: 'us-east-1b',
    InternetGatewayId: 'igw-dev1234567890abcdef0',
    S3VpcEndpointId: 'vpce-dev1234567890abcdef0',
    DynamoDBVpcEndpointId: 'vpce-dev1234567890abcdef1',
  },
  staging: {
    VpcId: 'vpc-staging1234567890abcdef0',
    VpcCidr: '10.1.0.0/16',
    PublicSubnet1Id: 'subnet-staging1234567890abcdef0',
    PublicSubnet2Id: 'subnet-staging1234567890abcdef1',
    PublicSubnet1Az: 'us-east-1a',
    PublicSubnet2Az: 'us-east-1b',
    InternetGatewayId: 'igw-staging1234567890abcdef0',
    S3VpcEndpointId: 'vpce-staging1234567890abcdef0',
    DynamoDBVpcEndpointId: 'vpce-staging1234567890abcdef1',
  },
  prod: {
    VpcId: 'vpc-prod1234567890abcdef0',
    VpcCidr: '10.2.0.0/16',
    PublicSubnet1Id: 'subnet-prod1234567890abcdef0',
    PublicSubnet2Id: 'su

# test/tap-stack.unit.test.ts
import * as cdk from 'aws-cdk-lib';
import { Match, Template } from 'aws-cdk-lib/assertions';
import { TapStack } from '../lib/tap-stack';

describe('TapStack', () => {
  let app: cdk.App;
  let stack: TapStack;
  let template: Template;
  const environmentSuffix = 'test';

  beforeEach(() => {
    app = new cdk.App();
    stack = new TapStack(app, `TapStack${environmentSuffix}`, {
      env: {
        account: '123456789012',
        region: 'us-east-1',
      },
      environmentSuffix,
    });
    template = Template.fromStack(stack);
  });

  describe('VPC Configuration', () => {
    test('creates VPC with correct CIDR block', () => {
      template.hasResourceProperties('AWS::EC2::VPC', {
        CidrBlock: '10.0.0.0/16',
        EnableDnsHostnames: true,
        EnableDnsSupport: true,
      });
    });

    test('VPC has correct tags', () => {
      const vpcs = template.findResources('AWS::EC2::VPC');
      const vpcResource = Object.values(vpcs)[0] as any;
      const tags = vpcResource.Properties.Tags;

      const nameTag = tags.find((tag: any) => tag.Key === 'Name');
      const envTag = tags.find((tag: any) => tag.Key === 'Environment');

      expect(nameTag).toBeDefi

# cdk.json
{
  "app": "npx ts-node --prefer-ts-exts bin/tap.ts",
  "watch": {
    "include": [
      "**"
    ],
    "exclude": [
      "README.md",
      "cdk*.json",
      "**/*.d.ts",
      "**/*.js",
      "tsconfig.json",
      "package*.json",
      "yarn.lock",
      "node_modules",
      "test"
    ]
  },
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/core:target-partitions": [
      "aws",
      "aws-cn"
    ],
    "@aws-cdk-containers/ecs-service-extensions:enableDefaultLogDriver": true,
    "@aws-cdk/aws-ec2:uniqueImdsv2TemplateName": true,
    "@aws-cdk/aws-ecs:arnFormatIncludesClusterName": true,
    "@aws-cdk/aws-iam:minimizePolicies": t
