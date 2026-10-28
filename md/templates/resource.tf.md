resource "{full_name}" "{resource_name}" {
  for_each = { for key, value in {variable_name} : key => value if value.resource_enabled{provider_filter} }

{provider_block}{required_arguments}

{optional_arguments}

{required_blocks}

{optional_blocks}

  lifecycle {
    prevent_destroy = false
  }
}
