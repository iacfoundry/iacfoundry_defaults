terraform {
  required_providers {
    {provider} = {
      source = "{developer}/{provider}"
      configuration_aliases = [
        {provider}.account,
        {provider}.workspace
      ]
    }
  }
}