# CloudFormation

- CloudFormation is AWS's infrastructure-as-code service. You describe infrastructure in a **template**, a YAML file (JSON also works).
- A template describes the **resources** to create, plus supporting sections: description, metadata, parameters, conditions and a few more. **`Resources` is the only required section.**
- CloudFormation turns a template into a **stack**. The stack holds **logical resources**, one per entry in `Resources`. CloudFormation then creates a **physical resource** in AWS for each logical one.

## Template

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Bucket for blog images

Parameters:            # inputs supplied when creating the stack
  BucketName:
    Type: String

Conditions:            # create resources only when a condition holds
  IsProd: !Equals [!Ref "AWS::AccountId", "111122223333"]

Resources:             # the only required section
  ImagesBucket:        # logical ID
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
```

## Template → stack → physical resources

```
template (YAML)          stack                          AWS account
                         ┌──────────────────────┐
Resources:               │ logical resource     │      physical resource
  ImagesBucket: ───────► │   ImagesBucket       │ ───► S3 bucket "my-blog-images"
    Type: AWS::S3::Bucket│   AWS::S3::Bucket    │
                         └──────────────────────┘
```

- **Logical resource**: the entry in the template, named by its logical ID (`ImagesBucket`).
- **Physical resource**: the real thing in AWS, with its own ID or name (`my-blog-images`).
- CloudFormation acts on the physical resources only when you change the stack:
  - Create the stack and CloudFormation creates the physical resources.
  - Update the stack with a new template and it changes them to match.
  - Delete the stack and it deletes them.
- It doesn't watch them in between. A manual change in the console isn't reverted. The mismatch is called **drift**, and **drift detection** reports it.
- One template can create many stacks (e.g. dev and prod), each with its own physical resources.
